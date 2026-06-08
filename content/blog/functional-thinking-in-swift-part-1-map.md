+++
title = "Functional Thinking in Swift: Part 1 - Map"
description = "An introduction to Swift's map function and how it helps express transformations in a clear, functional style."
date = 2025-12-16

[taxonomies]
tags = ["ios", "swift", "functional-programming"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/functional-thinking-in-swift-part-1-map-80c3700da5c2"
+++

Often, when people talk about Functional Programming (FP), it can become very abstract or academic, and the practical essence gets lost. This is especially true if it's not a paradigm you are used to; it requires a different way of thinking compared to the standard imperative approaches usually taught to beginners.

Swift gives us small functional tools that we often use without realizing they are "functional." As Swift isn't a pure FP language, we aren't trying to rewrite our entire app using strict FP. We simply want to use the tools we are given to simplify our code.

I'd like to build up a toolkit of these functional Swift tools so we can leverage some powerful, compositional thinking without getting lost in abstract definitions.

## Why functional thinking?

Functional thinking isn't about writing "clever" code. It's about changing the way you think about code. Instead of focusing on steps and mutable state, you focus on data flowing through small, predictable transformations.

This shift makes your intentions clearer, reduces hidden complexity, and leads to code that behaves the same way every time you run it. In other words: fewer surprises, fewer accidental edge cases, and a more deterministic system overall. Even the simplest functional tools, like `map`, start training your brain to express what should happen rather than how to make it happen, and that mindset scales beautifully as your codebase grows.

## The Map function

At its core, `map` just means:

Take a container of items. Transform the items. Return a new container of the transformed items.

That's it.

Let's look at a super small example first.

The imperative version:

```swift
let numbers = [1, 2, 3]
var doubled: [Int] = []

for i in numbers {
  doubled.append(i * 2)
}
// doubled = [2, 4, 6]
```

The functional equivalent using `map`:

```swift
let doubled = [1, 2, 3].map { $0 * 2 }
// doubled = [2, 4, 6]
```

Here, we have taken a container, `Array`, of items, `Int`, and transformed them by doubling their value into a new container, `Array`, of items, `Int`.

Rather than describing the steps to get there, we have described what we want the output to be: "An array where each value is double the value of the source array."

## Why does this matter?

`map` has done a few really nice things for us here:

- Declarative Style: We describe what we want rather than how to create it. No counters, no appending, and no manual iteration.
- Immutability: There is no mutation of values. Since many Swift containers are value types (copy-on-write), avoiding mutation gives us better clarity, thread safety, and predictability.
- Clearer Intent: "Transform these values" is much easier to read at a glance than "iterate over these values and accumulate them into a new variable."
- Safety: We don't have to worry about what to do if the container is empty. `map` handles empty containers gracefully.

## It works across many types

The definition of `map` doesn't talk about `Array`; it talks about "containers," and there are many different types of containers in Swift. All of them are mappable.

In every case, the function does the same thing: transform the contents of the container and return a new container with the new contents.

### Optional

Because `Optional` is a container, it contains either some value or none, we can map it. We get back another `Optional` of the transformed type.

```swift
let title: String? = "Hello, World!"
let titleLength = title.map { $0.count }
// titleLength is Int? (13)

let name: String? = nil
let nameLength = name.map(\.count)
// nameLength is nil
```

Here, `map` transforms the contents without changing the type of the container. If the value is `nil`, we simply get `nil` back. This saves a lot of `if let` unwrapping and safety checking.

### Result

`Result` is also a container. It contains either a `.success` value or a `.failure` error. `map` will transform the `Success` value if it exists, leaving the `Failure` case untouched.

```swift
let result: Result<User, any Error> = .success(userData)
let userName = result.map { $0.name }
// userName is Result<String, any Error> with the .success case.

let result: Result<User, any Error> = .failure(error)
let userAge = result.map(\.age)
// userAge is Result<Int, any Error> with the .failure(error) case.
```

## A real-world example

As a more "real" example, let's imagine a network call to a weather API that we want to call and then map into our view models.

```swift
struct WeatherCityDTO: Decodable {
  let id: UUID
  let city: String
  let humidity: Double?
  let temperatureCelsius: Double
}

struct CityViewModel {
  let name: String
  let humidityTitle: String?
  let tempTitle: String
}

func fetchCityWeather() async throws -> [CityViewModel] {
  try await networkClient.fetchCities()
    .map { cityDTO in
      CityViewModel(
        name: cityDTO.city,
        // Using map on the Optional property inside the Array map.
        humidityTitle: cityDTO.humidity.map { humidityDescription(from: $0) },
        tempTitle: "\(cityDTO.temperatureCelsius.rounded())°C"
      )
    }
}
```

We've used `map` to take an array of `WeatherCityDTO` from a network call and transformed it into an array of our own view models. We also used `map` to optionally create a `humidityTitle` if the humidity exists in the DTO.

All without mutating an array or creating a single `for` loop.

## The Trap: When Map Isn't Enough

`map` is fantastic when you have a 1-to-1 transformation. You put one thing in, you get one transformed thing out.

But what happens if your transformation also returns a container?

Imagine you have an array of `String` identifiers, and you want to turn them into URLs:

```swift
let strings = ["https://www.apple.com", "broken-link"]
let urls = strings.map { URL(string: $0) }
// Result: [Optional<URL>, Optional<URL>]
```

Because `URL(string:)` returns an optional, `map` wraps that optional inside the array. You end up with an array of optionals, `[URL?]`, which is usually annoying to work with. You technically have a container inside a container.

Or, what if you want to map over a list of user IDs and fetch their recent posts?

```swift
let userIDs = [1, 2, 3]
let posts = userIDs.map { fetchPosts(for: $0) }
// Result: [[Post], [Post], [Post]]
```

You end up with an array of arrays.

This is where `map` hits its limit. When the shape of your data starts getting nested, optionals inside arrays, or arrays inside arrays, `map` can't flatten that structure for you.

## Next time

This limitation is exactly where our next set of tools comes in. In the next part, we will look at `compactMap` and `flatMap` to see how we can handle these nested containers gracefully.

We will also look at `filter` and `reduce`, and eventually, how to compose all these simple tools together to perform powerful data transformations with readable, declarative code.
