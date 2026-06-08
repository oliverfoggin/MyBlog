+++
title = "Leveraging custom Reducers to manage actions in your TCA app"
description = "How to use a custom NestedAction reducer to keep nested TCA actions organised while still returning top-level effects."
date = 2023-12-19

[taxonomies]
tags = ["ios", "ios-development", "swift", "app-architecture", "tca"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/leveraging-custom-reducers-to-manage-actions-in-your-tca-app-d4c71719a1f6"
+++

As your TCA app grows, each feature accumulates more and more actions that it has to manage.

This could be UI actions, internal actions, child feature actions, delegate actions, etc.

```swift
enum Action {
  case onAppear
  case task
  case buttonTapped
  case closeButtonTapped
  case downloadResponse(Result<UserInfo>)
  case userProfile(UserProfile.State)
  case favourites(Favourites.State)
  case dismissFeature
}
```

After a while it becomes difficult to know which actions are which and we need a way to separate them. A widely used pattern is to create nested actions that hold onto only the actions relevant to it.

```swift
enum Action {
  case view(ViewAction)
  case internal(InternalAction)
  case destination(PresentationAction<Destination>)
  case delegate(DelegateAction)

  enum ViewAction {
    case onAppear
    case task
    case buttonTapped
    case closeButtonTapped
  }

  enum InternalAction {
    case downloadResponse(Result<UserInfo>)
  }

  enum Destination {
    case userProfile(UserProfile.State)
    case favourites(Favourites.State)
  }

  enum DelegateAction {
    case dismissFeature
  }
}
```

This gives us nice separation of the actions, but it makes it more difficult to organise the reducer body.

```swift
Reduce { state, action in
  switch action {
  case .view(.onAppear): ...
  case .view(.task): ...
  case .internal(.downloadResponse): ...
  case .destination(.presented(.userProfile)): ...
  case .delegate: ...
  case .view: ...
  }
}
```

It's very easy to get these in a muddle, lose exhaustivity, or duplicate logic unnecessarily.

But there is a way to make this much easier for yourself.

The `Reducer` type can be a bit daunting when you begin looking into how it works, but a lot of the custom reducers provided by the TCA package use a similar pattern. By looking at that and duplicating what we see, we can begin to put together an idea of how we might improve the situation here.

Wouldn't it be nice to create something that allowed us to spawn off a separate `Reducer` just for a particular sub-action?

Something like `NestedAction(\.view)` that would give us a body in which we just have to deal with the `ViewAction` actions without worrying about the others.

At first this seems similar to what `Scope` does, but in our case the problem is a bit more subtle. We're processing the `ViewAction`, but then we want to return an `Effect` that deals with the top-level `Action` enum. If we don't do this then we wouldn't be able to do something like this from the view action reducer:

```swift
return .send(.delegate(.dismissFeature))
```

Which is essential if we want our app to continue working. And also, we still want to be able to mutate the regular `State` without having to scope it.

So, we'll need something that is generic over `State`, `Action`, and `ChildAction`, and provides a way to scope into the child action.

This is what I came up with:

```swift
// 1.
@Reducer
public struct NestedAction<State, Action, ChildAction> {
  // 2.
  @usableFromInline
  let toChildAction: AnyCasePath<Action, ChildAction>

  // 3.
  @usableFromInline
  let toEffect: (inout State, ChildAction) -> Effect<Action>

  @inlinable
  public init(
    _ toChildAction: CaseKeyPath<Action, ChildAction>,
    toEffect: @escaping (inout State, ChildAction) -> Effect<Action>
  ) {
    self.toChildAction = AnyCasePath(toChildAction)
    self.toEffect = toEffect
  }

  public func reduce(into state: inout State, action: Action) -> Effect<Action> {
    // 4.
    guard let childAction = self.toChildAction.extract(from: action) else {
      return .none
    }

    // 5.
    return toEffect(&state, childAction)
  }
}
```

Let's unwrap what it's doing:

1. We define our `NestedAction` reducer and make it generic over `State`, `Action`, and `ChildAction`.
2. We have a property of a case path which tells us how to get from an `Action` to the relevant `ChildAction`.
3. Finally, we have a closure which takes in `State` and `ChildAction` and returns an `Effect<Action>`. This is the subtle difference between this and the `Reduce` reducer, as that will have a function from `State`, `Action` to `Effect<Action>`.
4. In the body of our `NestedAction` reducer we first check to see if the action coming in is one that we're interested in. We do this by trying to extract the `ChildAction` from the `Action` using the case path. If that doesn't work then we're not interested, so we just return a `.none` effect.
5. If the child action can be extracted then we pass it and the state into the closure and return any `Effect` that comes from that.

Now that we've created our new reducer, let's put it into use.

Let's deal with the view actions:

```swift
// We create a new NestedAction reducer just for the view action.
NestedAction(\.view) { state, action in
  switch action {
  case .onAppear:
    state.text = "Hello, user!"
    return .none

  case .task:
    return .none

  case .buttonTapped:
    return .run { send in
      let userInfoResult = await clientDependency.downloadUserInfo()
      await send(.internal(.downloadResponse(userInfoResult)))
    }

  case .closeButtonTapped:
    return .send(.delegate(.dismissFeature))
  }
}
```

For the other actions you can create separate `NestedAction` reducers to deal with them. The body of the reducer will look something like this. And using Xcode's folding ribbon it literally looks just like this:

```swift
var body: some ReducerOf<Self> {
  NestedAction(\.view) { ... }

  NestedAction(\.internal) { ... }

  NestedAction(\.delegate) { ... }

  NestedAction(\.destination) { ... }
}
```

Using this we could potentially mutate state in any of these reducers, but we still might want to use `ifLet` reducers or `onChange` reducers to add onto these. Especially with the `onChange` reducer, we don't know where a particular property might change, so which reducer do we add it to?

Thankfully TCA provides us with `CombineReducers` for free, which we can make use of here. So, adding an `onChange` reducer to the above might look like this:

```swift
var body: some ReducerOf<Self> {
  CombineReducers {
    NestedAction(\.view) { ... }

    NestedAction(\.internal) { ... }

    NestedAction(\.delegate) { ... }

    NestedAction(\.destination) { ... }
  }
  .onChange(of: \.text) { oldValue, newValue in ... }
  .ifLet(...) { ... }
}
```

This makes sure that wherever state is changed it will be caught in the `onChange` reducer. Similarly, it makes sure that the `ifLet` reducer is run before any of ours so that we can't invalidate the state in a way that breaks the app.

Reducers are really powerful in TCA, and by taking common problems and wrapping the logic into a new `Reducer`, we can really improve the look of our code and make it much more pleasurable to work with and maintain.

What new reducers have you created in your own TCA projects? I'd love to hear about them.
