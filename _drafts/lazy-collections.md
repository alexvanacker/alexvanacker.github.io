---
layout     : posts
title      : "debugging Lazy Eclipse Collections"
excerpt    : "A blog post about Lazy Collections, Eclipse Collections, and debugging"
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

## Lazy collections with EclipseCollections



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
`oldSize + index > oldSize + newSize - 1`.
Read that again, then think about it.

This means that `index > newSize - 1`.
And what is index? Well, it should be contained between `0` and `newSize - 1` (included) since it's the index on `elements` and `newSize` is `elements`'s number of items. Or mathematically put
```
 0 <= index <= newSize - 1

```

Simply put, the number of elements counted at the definition of `newSize` does not match the number of elements when we iterate on them.

## Putting it together with the previous code


## The final question


# Final considerations




[^newWithAll]: Taken from the source code of [AbstractImmutableList](https://github.com/eclipse/eclipse-collections/blob/master/eclipse-collections/src/main/java/org/eclipse/collections/impl/list/immutable/AbstractImmutableList.java)
