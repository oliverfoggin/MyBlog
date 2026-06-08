+++
title = "Shared data in a TCA app"
description = "How to manage shared app data in The Composable Architecture using a dependency that exposes an AsyncStream."
date = 2023-12-13

[taxonomies]
tags = ["ios", "ios-development", "swift", "app-architecture", "tca"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/shared-data-in-a-tca-app-9b16e14b75cd"
+++

A question that I come across a lot from people using TCA to help structure their apps is:

> How do you manage shared data in a large TCA app?

The initial thought process for many, myself included, is to somehow store shared data in the state of the root reducer and synchronise changes up and down the stack when it changes. So that children can make changes and it's updated back to the root. And then the root propagates these changes to all the children.

This very quickly becomes hugely complicated and there is a much more elegant way using dependencies.

Instead of using the root reducer to store our shared data we will create a dependency to store it for us. This dependency will expose an `AsyncStream` of the type that we want to store. For a very basic example let's create a shared number that all parts of our app can access and update.

Let's create our dependency:

```swift
@DependencyClient
struct NumberStore {
  var value: @Sendable () async -> AsyncStream<Int>
  var updateValue: @Sendable (Int) async -> Void
}
```

This is the signature of our dependency. It provides a way to get a stream of values that we should be able to subscribe to, and a way for us to update the current value.

With this we can begin to imagine what our reducer will look like. It will need:

- State to hold onto the number
- The `NumberStore` dependency
- An action to subscribe to the dependency
- An action to update the state

```swift
@Reducer
struct SomeFeature {
  struct State {
    var number: Int = 0
  }

  enum Action {
    case task // the convention is to use the task name to mirror the view task
    case newNumberReceived(Int)
  }

  @Dependency(\.numberStore) var numberStore

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .task:
        return .run { send in
          for await value in await numberStore.value() {
            await send(.newNumberReceived(value))
          }
        }

      case .newNumberReceived(let value):
        state.number = value
        return .none
      }
    }
  }
}
```

That's our basic reducer that subscribes to the updates. Now we need to implement the dependency.

We'll first create a class to hold onto the value for us.

```swift
class NumberStorage {
  @Published var value: Int = 0
}
```

Then our dependency becomes very simple:

```swift
extension NumberStore {
  static var liveValue: Self {
    let storage = NumberStorage()

    return .init(
      value: {
        storage.$value
          .streamValues
      },
      updateValue: {
        storage.value = $0
      }
    )
  }
}
```

Where we have defined `streamValues` as an extension on `Publisher`:

```swift
public extension Publisher where Self.Failure == Never {
  var streamValues: AsyncStream<Output> {
    AsyncStream(bufferingPolicy: .bufferingNewest(5)) { continuation in
      let cancellable = self.sink { continuation.yield($0) }
      continuation.onTermination = { _ in
        cancellable.cancel()
      }
    }
  }
}
```

With all this in place any reducer in the app can now subscribe to a stream of values held by the `NumberStore` dependency.

Now that we've done that we could also create an action to update the value:

```swift
case updateValueButtonTapped(Int)
```

And then handle it in the reducer:

```swift
case .updateValueButtonTapped(let value):
  return .run { _ in
    await numberStore.updateValue(value)
  }
```

That's it for updating the value. This will trigger the update in the dependency which will then push the new value out to any reducers that are listening out for it.

Finally we need to write the view to display the count.

```swift
struct NumberView: View {
  let store: StoreOf<SomeFeature>

  var body: some View {
    WithViewStore(store, observe: { $0 }) { viewStore in
      Text("\(viewStore.number)")
        .task { await store.send(.task).finish() }

      Button {
        store.send(.updateValueButtonTapped(viewStore.number + 1))
      } label: {
        Text("Add one")
      }
    }
  }
}
```

The use of `task` in the view will keep the subscription alive when the view is on screen and cancel it when the view disappears.

So this is all that we need. This view will update the value displayed when any part of the app updates the value.

Of course, this is only a very basic version and you may want to build in some security around race conditions, but this makes a lot of shared data very simple within a TCA app.
