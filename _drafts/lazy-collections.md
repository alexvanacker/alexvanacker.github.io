---
layout     : posts
title      : "debugging Lazy Eclipse Collections"
excerpt    : "A blog post about Lazy Collections, Eclipse Collections, and debugging"
draft      : true
date       : 2024-08-09 12:00:00
---

I have been a big fan of [Eclipse Collections](https://eclipse.dev/collections/) ever since our architect at Concord introduced this library to me. At the time, Java 9 wasn't out, so we didn't have immutable lists, which this library did. But more than that, it brought very _semantic_ methods for what we wanted to write as code, compared sometimes to the Stream API.

See the following code example written with Streams, which takes a collection of Animal objects, filters those that are not cats, then gets these cats owners' name.

```java
public static List<String> withStreams() {
    
    List<Animal> allAnimals =  List.of();
    return allAnimals.stream()
            .filter(animal -> !animal.isCat())
            .map(Animal::getOwnerName)
            .toList();
}
```


Now written with Eclipse Collections:

```java
    public List<String> withEC() {
    
        ImmutableList<Animal> allAnimals =  Lists.immutable.of();
        return allAnimals
                .reject(Animal::isCat)
                .collect(Animal::getOwnerName)
                .toList();
    }
```

_Spoiler: I find this second block much clearer_

But that's besides today's point. Eclipse Collections also brought with them past a certain version the notion of _Lazy collections_. I came upon a funny little bug at work that made me realize that, if not carefully used, this could blow up in your face. As most bugs do. Let's first see what these strange objects are.


# What are Lazy collections?

So, what are Lazy collections? Laziness comes from the functional programming world. Basically, these are collections that will not need to iterate on each of its element until a _final operation_ is applied. For example, see the pseudo-code below:

```java

myList.map(element -> slowCode(element))
    .take(5);
```

If we were to use an eager collection (i.e. _not_ lazy), we would apply the `slowCode` method on each element... only to take the result of the first 5!
Should your list be huge, or potentially infinite (think streams), then this would be quite the waste.

With lazy collections, any iteration operation that is not needed by a subsequent operation is deferred until that moment. So, this would push back any processing and temporary object creation until these are actually needed.

If we take back the very basic example shown above, the `slowCode` would only happen for the five elements needed instead of the whole list.
What happens is:

```java
myList.take(5)
    .map(element-> slowCode(element))
```

## Lazy collections with EclipseCollections
To keep?


# So what was the issue?

We were working with chaining multiple calls that seemed innocent enough, putting the whole lists together as one in an `ImmutableList`.
Below is basically what we were doing (for obvious reasons, I'm not sharing our exact code):

```java

final Function<Integer, QueryResult> queryRequest = pageNbr -> queryService.query(pageNbr);

final firstResults = Lists.immutable.of(queryService(0));
final otherResults = Interval.fromTo(1, 10)
        .collect(queryRequest)
        .flatCollect(QueryResult::getResults); // This is a LazyIterable!
        

final wholeResults = firstResulst.newWithAll(otherResults); // Create a new immutable list with all the results
```

All looks fine here right? We actually don't call the query service until it's needed, i.e. when we'll create the final `wholeResults` list.

And then one beautiful August day, this stacktrace appeared in our errors:

```
java.lang.ArrayIndexOutOfBoundsException: Index 391 out of bounds for length 391

at org.eclipse.collections.impl.list.immutable.AbstractImmutableList.lambda$newWithAll$5e9f739d$1(AbstractImmutableList.java:207)
[... omitted for SOC2 compliance and brevity]

```

## Reading the code

When we open up this `newWithAll` method[^newWithAll] that decided to ruin my day, here is what we had:

```java
    @Override
    public ImmutableList<T> newWithAll(Iterable<? extends T> elements)
    {
        int oldSize = this.size();
        int newSize = Iterate.sizeOf(elements);
        T[] array = (T[]) new Object[oldSize + newSize];
        this.toArray(array);
        Iterate.forEachWithIndex(elements, (each, index) -> array[oldSize + index] = each);
        return Lists.immutable.with(array);
    }
```

How do we blow this up? The only way for the exception to happen would be for: 
$$ oldSize + index > oldSize + newSize - 1 $$


Read that again, then think about it.

This means that 

$$index > newSize - 1$$

And what is index? Well, it should be contained between `0` and `newSize - 1` (included) since it's the index on `elements` and `newSize` is `elements`'s number of items. Or mathematically put:

$$ 0 <= index <= newSize - 1 $$

Simply put, the number of elements counted at the definition of `newSize` does not match the number of elements when we iterate on them. Let us see how that is possible.

## Putting it together with the previous code

Remember the code before: `elements` we pass is a `LazyIterable`. And how is that `LazyIterable` built? We fill the elements using the function that runs a queryService on a given page number. In our case, it could very well happen that the query results were _not the same_ between two calls. Or, simply put, this is not a pure function[^pureFunction].

What we are sayingis that between the call to `Iterate.sizeOf` and `Iterate.forEachWithIndex`, the results returned by `queryService` could be different, leading to a different number of items in elements. And back to that Exception blowing up in our face.

How can we show this easily? A simply Java unit test can reproduce it, but even as simple is actually expecting the same sizes between two calls:

```java

    /**
     * A function returning a random number of elements. 
     **/
    Function<Integer, ImmutableList<Integer>> random = integer -> {
		final var size = (int) (Math.random() * 100);
		return Interval.oneTo(size).toImmutableList();
	};

    @RepeatedTest(10)
	void sizeTest() {
		final var items = Interval.fromTo(1, 10)
		        .collect(i -> i)
		        .flatCollect(i -> random.apply(i)).toImmutableList();
		Assertions.assertThat(items.size())
		        .isEqualTo(items.size());
	}
```

If you run this in your favorite IDE, you'll see that this one fails numerously out of the ten attemps we are giving it.

## The final question

After that I was happy at first, but something bugged me still; I was under the impression that if we call any final operation (typically a forEach) on a Lazy collection, then it would become Eager somehow (i.e. be evaluated) and so would not change anymore. 
And what really ticked me off is when 

# Final considerations

So how could we have avoided this?

// TODO Talk about simply calling the toList at the end of the call
// TODO Talk about using a reduce? (maybe too much)




[^newWithAll]: Taken from the source code of [AbstractImmutableList](https://github.com/eclipse/eclipse-collections/blob/master/eclipse-collections/src/main/java/org/eclipse/collections/impl/list/immutable/AbstractImmutableList.java)
[^pureFunction]: A function that given the same input always returns the same output, without any side effects. See [Wikipedia](https://en.wikipedia.org/wiki/Pure_function). I don't have time to get into the details.
