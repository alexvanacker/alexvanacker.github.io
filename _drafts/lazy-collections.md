---
layout     : posts
title      : "debugging Lazy Eclipse Collections"
excerpt    : "A blog post about Lazy Collections, Eclipse Collections, and debugging"
draft      : true
date       : 2024-08-09 12:00:00
---

In this post I am going to talk about a Java library named Eclipse Collections, Lazy Collections and how using it unwisely lead to a nice bug recently. 

I have been a big fan of [Eclipse Collections](https://eclipse.dev/collections/) ever since our architect at Concord introduced this library to me. At the time, Java 9 wasn't out, so we didn't have immutable lists, which this library did. But more than that, it brought very _semantic_ methods, such as `reject`, for what we wanted to write as code, compared to the Streams API.

See the following code example written with Streams, which takes a collection of Animal objects, filters those that are not cats, then gets these cats owners' name.

```java
public List<String> withStreams() {
    List<Animal> allAnimals =  List.of(/*...*/);
    return allAnimals.stream()
            .filter(animal -> !animal.isCat())
            .map(Animal::getOwnerName)
            .toList();
}
```


Now written with Eclipse Collections:

```java
public List<String> withEC() {
    ImmutableList<Animal> allAnimals =  Lists.immutable.of(/**...**/);
    return allAnimals
            .reject(Animal::isCat)
            .collect(Animal::getOwnerName)
            .toList();
    }
```

_Spoiler: I find this second block much clearer_

But that's besides today's point. Eclipse Collections also brought with them the notion of _Lazy collections_. I came upon a funny little bug at work that made me realize that, if not carefully used, these could blow up in your face. As most bugs do. Let's first see what these strange objects are.


# What are Lazy collections?

So, what are Lazy collections? Laziness comes from the functional programming world. Basically, these are collections that will not need to iterate on each of its element until a _final operation_ is applied. For example, see the pseudo-code below:

```java

myList.map(element -> slowCode(element))
    .take(5);
```

If we were to use an eager collection (i.e. _not_ lazy), we would apply the `slowCode` method on each element... only to take the result of the first 5!
Should your list be huge, or potentially infinite (think streams), then this would be quite the waste.

With lazy collections, any iteration operation that is not needed by a following operation is deferred until that moment. So, this would push back any processing and temporary object creation until these are actually needed.

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

final Function<QueryParams, QueryResult> queryRequest = params -> queryService.query(params);

final ImmutableList<Results> firstResults = queryService(0).getResults();
final QueryParams queryParams = /*set by method arguments*/;
// otherResults is a Lazy collection (LazyIterable)
final otherResults = Interval.fromTo(1, 10)
        .collect(queryParams::withPageNbr)
        .collect(queryRequest)
        .flatCollect(QueryResult::getResults);
        
// Create  a new immutable list with all the results
final wholeResults = firstResulst.newWithAll(otherResults);
```

All looks fine here right? We actually don't call the `queryService` until it's needed, i.e. when we'll create the final `wholeResults` list.

And then, one beautiful August day, this stacktrace appeared in our errors:

```
java.lang.ArrayIndexOutOfBoundsException: Index 391 out of bounds for length 391

at org.eclipse.collections.impl.list.immutable.AbstractImmutableList.lambda$newWithAll$5e9f739d$1(AbstractImmutableList.java:207)
[... omitted for SOC2 compliance and brevity]

```

## Debugging, i.e. reading the code

When we open up this `newWithAll` method[^newWithAll] that decided to ruin my day, here is what we saw:

```java
@Override
public ImmutableList<T> newWithAll(Iterable<? extends T> elements) {
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

And what is index? Well, it should be contained between `0` and `newSize - 1` (included) since it's the index on `elements` and `newSize` is `elements` number of items. Or mathematically put:

$$ 0 <= index <= newSize - 1 $$

Headbanging aside, we are saying that the number of elements counted at the definition of `newSize` does not match the number of elements when we iterate on them. Let us see how that is possible.

## Putting it together with the previous code

Remember the code before: `elements` we pass is a `LazyIterable`. And how is that `LazyIterable` built? We fill the elements using the function that runs a queryService on a given page number. In our case, it could very well happen that the query results were _not the same_ between two calls. Or, simply put, this is not a pure function[^pureFunction].

What we are saying is that between the call to `Iterate.sizeOf` and `Iterate.forEachWithIndex`, the results returned by `queryService` could be different, leading to a different number of items in elements. And back to that Exception blowing up in our face.

How can we show this easily? A Java unit test can reproduce it, but even as simple is actually expecting the same sizes between two calls:

// TODO REWRITE THIS WITH REFERENCE TO GITHUB PROJECT

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
                .flatCollect(i -> random.apply(i))
                .toImmutableList();
        Assertions.assertThat(items.size())
                .isEqualTo(items.size());
    }
```
_Note: if you don't know flatCollect, it takes multiple collections to turn it into one, flattening said collections._


If you run this in your favorite IDE, you'll see that this one fails numerously out of the ten attemps we are giving it.

## The final question

After that I was happy at first, I found the bug cause!

But something bugged me still; I was under the impression that if we call any final operation (typically a forEach) on a Lazy collection, then it would become Eager somehow (i.e. be evaluated) and so would not change anymore.
And what really ticked me off is when I started going down the rabbit hole of how was implemented `Iterate.sizeOf`. I'll spare you the numerous layers to go through, in the end, we reach this method in the `AbstractRichIterable` class:

```java
@Override
public int count(Predicate<? super T> predicate)
{
    CountProcedure<T> procedure = new CountProcedure<>(predicate);
    this.forEach(procedure);
    return procedure.getCount();
}
```

So, we do have a `forEach` that was applied to the Lazy collection, yet it still yielded different results. That is what bugged me.

But the reason I was stuck is because I thought once you processed a Lazy collection, then it was evaluated for all subsequent calls. And that is not the case.
With EclipseCollections, Lazy collections can be reused multiple times. **They are not like Streams**, which, once processed, are done for. Which is great, but must be remembered when used. Because in our specific case, if you think about it, we were running the `queryService` (and you can assume it was not a light operation) _twice_ per page number. "A performance bug" as my colleague stated in hindsight.

# Final considerations

So how could we have avoided this? First off, you want to use Lazy if you have more than one operation that would create use resources, such as creating more temporary objects or collections. We are targetting efficiency here. Which was the case. 

_However_, by joining all Lazy collections into an ImmutableList, we were making this laziness useless at that point. So instead of relying on Lazy collections,
we should have forced the evaluation as such: 

```java

final Function<Page, QueryResult> queryRequest = page -> queryService.query(page);

final ImmutableList<Results> firstResults = queryService(0).getResults();
final QueryParams queryParams = /*set by method arguments*/;
// otherResults is a Lazy collection (LazyIterable)
final otherResults = Interval.fromTo(1, 10)
        .collect(queryParams::withPageNbr)
        .collect(queryRequest)
        .flatCollect(QueryResult::getResults)
        .toList(); // <- the fix
        
// Create  a new immutable list with all the results
final wholeResults = firstResulst.newWithAll(otherResults);

```

Yes, it was that simple.


## Takeaways

//TODO Write this properly 
1. Debugging is done by reading the code
2. When you think the bug comes from the underlying library, it most often doesn't
3. Lazy does not mean magic, there are caveats.



[^newWithAll]: Taken from the source code of [AbstractImmutableList](https://github.com/eclipse/eclipse-collections/blob/master/eclipse-collections/src/main/java/org/eclipse/collections/impl/list/immutable/AbstractImmutableList.java)
[^pureFunction]: A function that given the same input always returns the same output, without any side effects. See [Wikipedia](https://en.wikipedia.org/wiki/Pure_function). I don't have time to get into the details.
