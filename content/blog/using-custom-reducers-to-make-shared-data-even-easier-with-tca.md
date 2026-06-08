+++
title = "Using custom reducers to make shared data even easier with TCA"
description = "How a custom SubscriberReducer can remove repeated AsyncStream subscription boilerplate from TCA reducers."
date = 2024-01-02

[taxonomies]
tags = ["ios", "ios-development", "swift", "app-architecture", "tca"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/using-custom-reducers-to-make-shared-data-even-easier-with-tca-c9b18124bc5c"
+++

You can jump straight into the code here: [swift-composable-subscriber](https://github.com/oliverfoggin/swift-composable-subscriber).

In my recent article, [Shared data in a TCA app](@/blog/shared-data-in-a-tca-app.md), I detailed how to use dependencies to store shared data that can be subscribed to from anywhere in your app.

This means that anywhere in your app a reducer can return a `run` effect from its body, loop on the values from an `AsyncStream`, and return some other action to deal with the updated values.

```swift
case .task:
  return .run { send in
    for await value in await someDependency.valueStream() {
      await send(.newValueReceived(value))
    }
  }
```

This makes it really easy to subscribe to any stream of data in your app and even keeps your reducers in sync when the data is updated.

I then received a question on Mastodon from Ben Lings:

> Great stuff! This pattern has been very useful in the app we are building. I've been considering how to encapsulate accessing this sort of dependency in a higher order reducer, to reduce the task-foreach-send boilerplate. Any thoughts on that?

[View the original Mastodon post](https://hachyderm.io/@ben_lings/111578761999610292).

And I was thinking about this in the app I'm working on currently. There are a few reducers that rely on multiple streams of data. This very quickly started to flood the reducer body with these same lines over and over again. We needed a way to refactor this out to make it easier to maintain whilst still giving us the control that we have here.

This is where a custom reducer can come in to help us.

The problem we are trying to solve can be described as:

- When an `Action` occurs
- We should subscribe to some `AsyncStream`
- And when it receives a new value we should run some other `Action`

We can create this by creating our own custom reducer.

I created a `SubscriberReducer` that is generic over:

- `Parent: Reducer`, because it needs to know about the `Action` and `State` of the parent.
- `TriggerAction`, a specific action that will be used to trigger the subscription.
- `StreamElement`, the type of element that the `AsyncStream` will yield.
- `Value`, the type of the value that we want to handle. This allows us to provide a transform if necessary.

By doing this we can create several helper functions like:

```swift
.subscribe(to: myDependency.valueStream, on: \.task, with: \.newValueReceived)
```

This takes the five lines above and reduces it to a single line. And it allows us to add multiple subscriptions very easily.

If the `AsyncStream` type matches the action then we can just do this:

```swift
Reduce { state, action in
  // This is the regular reducer.
}
.subscribe(to: myDependency.valueStream, on: \.task, with: \.newValueReceived)
```

If the type of the `AsyncStream` doesn't match the action then you can provide a simple transform:

```swift
Reduce { state, action in
  // This is the regular reducer.
}
.subscribe(to: myDependency.intStream, on: \.task, with: \.newStringReceived) { intValue in
  "\(intValue)"
}
```

And even if we have more complex logic that we need to run, we can do that also:

```swift
Reduce {
  // This is the regular reducer.
}
.subscribe(
  to: myDependency.stream,
  on: \.some.trigger.action
) { send, streamElement in
  await send(.responseAction)
  await otherDependency.doSomethingElse(with: streamElement)
}
```

This has been great for us as it has allowed us to greatly simplify the code involved when we have multiple subscriptions firing off. And it makes it much easier to read also:

```swift
Reduce { state, action in
  // This is the regular reducer.
}
.subscribe(to: myDependency.userStream, on: \.task, with: \.newUserDataReceived)
.subscribe(to: myDependency.chatStream, on: \.task, with: \.newChatMessageReceived)
.subscribe(to: myDependency.reactionStream, on: \.task, with: \.newReactionReceived) {
  Reaction(payload: $0)
}
```

Custom reducers can really help shift a lot of the cognitive load of your app so that you don't have to worry about it at code time and it makes for easier reading and understanding too.

What custom reducers have you been able to create to help you offload repetitive work and reduce cognitive load?

I hope this has helped you. You can find my code and SPM package on GitHub: [swift-composable-subscriber](https://github.com/oliverfoggin/swift-composable-subscriber).
