---
layout: post
title: "Thoughts on defensive programming"
date: 2026-1-25 13:00:00 +0900
categories: engineering
lang: en
---

One of the few programming books I've read is _Effective Java_ by Joshua Bloch. Although it's a book about Java, not all the advice it gives is Java-specific, and it changed the way I think about programming in general. _Effective Java_ introduced me to the idea of "invariants," in particular "class invariants." At the time, it wasn't obvious to me that it's absolutely okay to _not_ defend against object states that are guaranteed impossible by invariants. One of the code snippets in the book presents an example of this:

```java
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

Strictly speaking, `type.cast` can throw an exception if the argument is not the correct type, but we know that `getFavorite` can never do this since all key-value pairs of `favorites` maintain an invariant: the key is a type token, and the value is an instance of that type token. Hence, `type.cast` is favored over a type-erased, non-throwing cast `(T)` since any bug that would violate the above invariant would cause an exception thrown by the class, rather than by the caller. This is a toy example, but it demonstrates what I have observed in practice: projects I have worked on where defensive programming was _discouraged_ had fewer bugs, because bugs manifest sooner, closer to where they actually are.

While "class invariants" are matter-of-fact, I have not heard an analogous expression used when talking about programs consisting of many classes or systems containing of many programs. But that doesn't mean they don't exist - the vocabulary simply changes to "method contracts," "API contracts," "architecture," or "agreements" which all can refer to state invariants at a larger scale.

An example still remains fresh in my memory. One of the projects I've worked on was heavily dependent on blob of data that configured the application. The data consisted of many pieces which could be referred to by ID, and such IDs were spread throughout the system - in the data itself and in our databases. However, as a team we had agreed that these references could _never_ be broken, and consequently, most application code never entertained the possibility that an ID could refer to missing data. I say _most_ because some parts of the system were dedicated to maintaining that invariant, so that the rest of the system did not have to worry about it.

As a result we could write code like this:

```java
class App {
    private final AppData data;

    // ...

    void doSomethingWithData() {
        var a = data.getThingA();
        var b = data.getThingB();
        var c = b.getThingCByIdOfA(a.getId());
        // do something with c
        // ...
    }
}
```

Instead of code like this:

```java
class App {
    private final AppData data;

    // ...

    void doSomethingWithData() {
        var a = data.getThingA();
        var b = data.getThingB();
        var c = b.getThingCByIdOfA(a.getId());
        if (c == null) {
            // no invariant, so entertain possibility of null
        } else {
            // do what the app is actually supposed to do
        }
    }
}
```

Which doesn't seem _that_ bad but when multiplied hundreds of times by a large team of developers, it can get _pretty bad_:

- Hundreds of places in the code are dedicated to working around a possibility that could have instead been eliminated by an invariant.
- If the chosen failure mode is not visible, inconsistent data may go unnoticed.
- Time is wasted on implementing a failure mode in the first place, resulting in a lot more code that is less readable.

This isn't an argument against defensive programming, it's an argument against defensive programming that could instead be replaced by invariants. The problem can then be reduced to choosing and enforcing invariants at the system level, which is not just a software problem, but (I believe) an organizational and cultural one.
