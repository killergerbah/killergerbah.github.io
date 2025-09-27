---
layout: post
title: "Patterns for event subscription"
date: 2025-9-27 18:21:00 +0900
categories: engineering
lang: en
---

There are a lot of ways to implement event subscription. Below are examples that I've seen in the wild.

## Spring's `@EventListener`

Spring provides a dead-simple annotation-based API for event subscription. There's no need to think about unsubscription, because Spring manages everything.

```java
@Service
public final class ListeningSpringBean {
    // ...

    @EventListener
    public void onSomeEvent(SomeEvent event) {
        // Do stuff with event
    }
}
```

## React's useEffect

React can subscribe functions to state changes. Unsubscription of the effect is handled by React, because the lifecycle of `Component` is already managed by React.

```typescript
const Component = () => {
  const [flag, setFlag] = useState<boolean>(false);

  useEffect(() => {
    // Do stuff when 'flag' changes
  }, [flag]);

  // ...
};
```

Incidentally, the return value of the function passed to `useEffect` will be run before the next execution of the effect, so we can use it to manage subscriptions to other events.

```typescript
const Component = () => {
  useEffect(() => {
    const listener = (event: MessageEvent) => {
      // Do stuff with event
    };
    document.addEventListener("message", listener);
    return () => document.removeEventListener("message", listener);
  }, []);

  // ...
};
```

Unfortunately we still need to call `document.removeEventListener` manually. However, this is hard to mess up since tear-down code is mandated by the API design.

## Unity Monobehaviour lifecycle methods

Unity `MonoBehaviour` lifecycle methods are automatically subscribed to lifecycle events.

```csharp
public class SomeComponent : MonoBehaviour {
    void OnEnable() {
        // Do stuff when the behavior is enabled
    }

    void OnDisable() {
        // Do stuff when the behavior is disabled
    }

    // ... and so on
}
```

## CSharp events

CSharp defines an event abstraction built into the language. When used in Unity code bases, it might look something like below.

```csharp
public class Publisher {
    public event Action OnSomethingHappened;

    // ...
}

public class SomeComponent : MonoBehaviour {

    // ...

    void OnEnable() {
        // Subscribe to OnSomethingHappened when this behavior is enabled
        _publisher.OnSomethingHappened += HandleSomethingHappened;
    }

    void OnDestroy() {
        // Unsubscribe when this behavior is destroyed
        _publisher.OnSomethingHappened -= HandleSomethingHappened;
    }

    void HandleSomethingHappened() {
        // Do stuff when something happens
    }
}
```

But there's a bug in this code: it's possible for `OnEnable` to be called many times, while `OnDestroy` can only be called once! This could be fixed by defensively unsubscribing in `OnEnable`.

```csharp
public class SomeComponent : MonoBehaviour {
    // ...

    void OnEnable() {
        // Ugly and defensive unsubscribe + subscribe
        _publisher.OnSomethingHappened -= HandleSomethingHappened;
        _publisher.OnSomethingHappened += HandleSomethingHappened;
    }

    // ...
}
```

# Observations

The above examples are ordered roughly in order of ease-of-use:

- With Spring's `@EventListener`, it's impossible to mis-manage event subscription because subscription it is already managed by Spring.
- On the other hand, subscriptions to CSharp events are extremely easy to mis-manage, because the subscription must be handled entirely by the developer, with care.

Granted, Spring's API is designed with a specific use-case in mind: singleton beans representing components of a usually stateless application, while CSharp events are a language-level feature that was designed to serve a variety of use-cases. However, that does not mean that we cannot learn from this comparison.

## Pitfalls of CSharp events

We can first consider why CSharp events are hard to use.

### A reference to the publisher is required

To subscribe to the event, we must first have a reference to the publishing `event`, creating a dependency on the publisher.

### It is possible to make a mistake during subscription

As demonstrated with the Unity example above, it is possible to over-subscribe to an `event`. Furthermore, the `event` cannot be easily inspected to avoid or debug this.

### It is possible to make a mistake during unsubscription

Unsubscription requires the same parameters as the original subscription:

- A reference to the `event`.
- A reference to the subscribing function.

Therefore, the developer must take care to supply the correct parameters during unsubscription.

## Advantages of frameworks

For the reasons above, I think it is preferable for developers working within an existing framework to use event abstractions provided by the framework. In fact, all of the problems of CSharp events do not exist with Spring's `@EventListener`:

- A reference to the publisher is not required.
- It's impossible to over-subscribe, and it's impossible to forget to unsubscribe, because Spring manages the subscription along with the bean's lifecycle.

## Advantages of alternative API design

Even if event subscription isn't already solved by the tools available to you, it's always possible to design your own event subscription API. Learning from the comparisons above, we propose some guidelines for designing such an API:

- If manual unsubscription is required, it should not require having to remember exactly what was subscribed.
- Subscribing to application-level events should not require specifying a specific publisher.

For example, instead of using CSharp events, one might write their own event publisher following the guidelines above:

```csharp
interface IEventPublisher
{
    /// <summary>
    /// Subscribes the callback <c>subscriber</c> to the event of type <c>TEvent</c>.
    /// </summary>
    /// <param name="subscriber">The subscribing function</param>
    /// <returns>
    /// An IDisposable that unsubscribes the subscriber from the event when disposed.
    /// </returns>
    IDisposable Subscribe<TEvent>(Action<TEvent> subscriber);

    /// <summary>
    /// Publishes an event of type <c>TEvent</c>
    /// </summary>
    void Publish<TEvent>(TEvent event);
}
```

or use an existing one.

# Why does this matter?

Given that it can take hours to solve just one bug caused by misusage of CSharp events, imagine the days or even weeks of time saved by simply offering a better API to developers. It is always preferable to solve problems before they happen - and designing or adopting great APIs in the early stages of a project is one way to do that.
