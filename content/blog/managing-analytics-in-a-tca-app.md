+++
title = "Managing Analytics in a TCA app"
description = "A composable way to keep analytics close to TCA feature logic without letting it crowd reducers or views."
date = 2023-12-14

[taxonomies]
tags = ["ios", "ios-development", "swift", "app-architecture", "tca", "analytics"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/managing-analytics-in-a-tca-app-10f6dec412fd"
+++

One of the core requirements of any enterprise app is analytics. It's essential if the company wants to know how their app is used, how they can improve it, and ultimately how they can maximise its profitability.

Integrating analytics into an app can be fairly easy using one of the many services that provide a way to log analytics with nice dashboards. Adding events to button presses, download results, or screen views can be very easy to do, but it can also take over the app and become invasive.

You can very quickly get into a state where the analytics in the app start to crowd the actual features of the app. Consistency doesn't exist as there are no real rules as to where the analytics should be done. And testing analytics becomes difficult when they are reported from button presses or scroll events in the view.

With TCA we have a well defined place to put all the logic of our apps: in the `Action` of the `Reducer`s.

Can we exploit this to make analytics just as well defined without also flooding the feature code?

My initial thought for analytics with TCA was to create a dependency that we can use to push our analytics through. In the reducer it looked something like this:

```swift
var body: ReducerOf<Self> {
  Reduce { state, action in
    switch action {
    case .buttonTapped:
      state.message = "Button tapped!"
      return .merge(
        .run { _ in
          await analyticsClient.send("ButtonTapped")
        },
        downloadEffect
      )
    }
  }
}
```

This works, but it doesn't solve the problems. The feature code of the app quickly gets overrun with analytics. In the example above the `buttonTapped` action is really there to start a `downloadEffect`, but it has to be merged with the analytics, making it harder to see what the app is actually doing.

It gets messy.

By leveraging the composability of TCA reducers, and creating our own reducer, we can create a much more elegant way of sending analytics that is just as powerful, but confined to its own dedicated space away from the feature code.

If you want to jump ahead and just get the code, I have created an SPM package that you can use: [swift-composable-analytics](https://github.com/oliverfoggin/swift-composable-analytics).

The core idea of this method is to still use a dependency to send analytics through, but to create a way to access that dependency through a dedicated reducer that does nothing other than send analytics. It means that the analytics code still remains close to the feature code, but just far enough away to not flood the feature code.

The end goal looks like this:

```swift
var body: some ReducerOf<Self> {
  // 1.
  AnalyticsReducer { state, action in
    // 2.
    switch action {
    case .buttonTapped:
      // 3.
      return .event(name: "ButtonTapped")
    }
  }

  Reduce { state, action in
    switch action {
    case .buttonTapped:
      state.message = "Button tapped!"
      // 4.
      return downloadEffect
    }
  }
}
```

1. We have a new `AnalyticsReducer` that gives us state and action just like any other reducer. But this time state is immutable. This means that our analytics code cannot affect the state of the app.
2. We have access to all the actions defined in the reducer, so we can send analytics for any event that can happen in the app. Whether it is the user tapping a button, the results of a successful download, or an error received from a service we interact with.
3. The `AnalyticsReducer` function just returns `AnalyticsData`, so it can't run any custom effects. It is completely isolated from the feature code.
4. The feature code of the app is unaffected. We just treat the feature code as the standard feature code. In this case, the button just returns the `downloadEffect` without needing to worry about analytics at all.

That's the plan, and all of this is done with a surprisingly small custom reducer:

```swift
import Foundation
import ComposableArchitecture

public struct AnalyticsReducer<State, Action>: Reducer {
  @usableFromInline
  let toAnalyticsData: (State, Action) -> AnalyticsData?

  @usableFromInline
  @Dependency(\.analyticsClient) var analyticsClient

  @inlinable
  public init(_ toAnalyticsData: @escaping (State, Action) -> AnalyticsData?) {
    self.init(toAnalyticsData: toAnalyticsData, internal: ())
  }

  @usableFromInline
  init(toAnalyticsData: @escaping (State, Action) -> AnalyticsData?, internal: Void) {
    self.toAnalyticsData = toAnalyticsData
  }

  @inlinable
  public func reduce(into state: inout State, action: Action) -> Effect<Action> {
    guard let analyticsData = toAnalyticsData(state, action) else {
      return .none
    }

    return .run { _ in
      analyticsClient.sendAnalytics(analyticsData)
    }
  }
}
```

This `AnalyticsReducer` requires a function from `(state, action) -> AnalyticsData?`, and everything else is done for you. If you return `nil` from the function then nothing happens. Only when you return the `AnalyticsData` does this access the `analyticsClient` and send the data to our analytics services.

Now we just need to create an `AnalyticsClient` to use to send events for us. By default in my package I have created a `.consoleLogger` analytics client that logs events out to the console. It looks like this:

```swift
extension AnalyticsClient {
  public static var consoleLogger: Self = .init(
    sendAnalytics: { analytics in
      print("[Analytics] ✅ \(analytics)")
    }
  )
}
```

It has a single function, `sendAnalytics`, that takes the data and logs it to the console.

To create our own `AnalyticsClient` that sends events to our own service we just need to implement this `sendAnalytics` function. Here is an example of a client that is written to send analytics to Firebase and Crashlytics:

```swift
import Firebase
import FirebaseCrashlytics

public extension AnalyticsClient {
  static var firebaseClient: Self {
    return .init(
      sendAnalytics: { analyticsData in
        switch analyticsData {
        case let .event(name: name, properties: properties):
          Firebase.Analytics.logEvent(name, parameters: properties)

        case .userId(let id):
          Firebase.Analytics.setUserID(id)
          Crashlytics.crashlytics().setUserID(id)

        case let .userProperty(name: name, value: value):
          Firebase.Analytics.setUserProperty(value, forName: name)

        case .screen(name: let name):
          Firebase.Analytics.logEvent(AnalyticsEventScreenView, parameters: [
            AnalyticsParameterScreenName: name
          ])

        case .error(let error):
          Crashlytics.crashlytics().record(error: error)
        }
      }
    )
  }
}
```

We extend `AnalyticsClient` to provide a `.firebaseClient`, and in the `sendAnalytics` function we just inspect the data that comes in and send it to the relevant Firebase or Crashlytics service.

To start using this we just need to inject the dependency when we create the root `Store` of the app:

```swift
let rootStore = Store(RootReducer.State()) {
  RootReducer()
} withDependencies: {
  $0.analyticsClient = .merge(.consoleLogger, .firebaseClient)
}
```

This is all we need now to start sending analytics from anywhere in the app, and we can use `merge` to add multiple analytics services to our app.

Finally we come to testing. Because all of the analytics of our app are sent via actions in a reducer it means that our tests have access to all of them.

And we can provide a function to expect analytics data that is really powerful for testing. In our test we can tell the `TestStore` that we expect to receive certain analytics data, and if we don't receive that the test will fail.

```swift
func testButtonTapped() async throws {
  let store = TestStore(RootReducer.State()) {
    RootReducer()
  } withDependencies: {
    $0.analyticsClient.expect(.event("ButtonTapped"))
  }

  store.send(.buttonTapped)
}
```

This test is exhaustive too.

If you don't expect an event and then an event is sent, it will fail. If you expect an event and it doesn't get sent, it will fail. If the expected event is different from the sent event, it will fail.

So the test really does cover every eventuality.

I have found this to be a really effective way to manage and send analytics throughout the apps I've worked on. Adding a new analytics service is just a case of creating a new client and injecting it at the root. All of the analytics throughout the app then start getting sent to the new service.

I hope this has given you a brief insight into how to leverage reducers and actions. This is only one use case I have created, but there are a lot more ways of extending TCA to provide benefits like this.

You can get my SPM package [swift-composable-analytics](https://github.com/oliverfoggin/swift-composable-analytics) on GitHub.
