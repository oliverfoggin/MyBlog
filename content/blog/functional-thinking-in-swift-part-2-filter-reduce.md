+++
title = "Functional Thinking in Swift: Part 2 - Filter & Reduce"
description = "A practical look at Swift's filter and reduce functions, and how to compose them with map into readable data pipelines."
date = 2025-12-23

[taxonomies]
tags = ["ios", "swift", "functional-programming"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/functional-thinking-in-swift-part-2-filter-reduce-ee388b059b5a"
+++

In [Part 1](@/blog/functional-thinking-in-swift-part-1-map.md) we looked at `map` and how it helps us transform data. But transforming data is only half the battle; often, we need to narrow down our datasets or combine them into a single value. This is where `filter` and `reduce` come in.

We will look at how `filter` and `reduce` work on their own and what they do, but then also how we can compose these functions together to unlock the "super power" of functional thinking in Swift.

## Filter: The Gatekeeper

The `filter` function does exactly what it says on the tin: it takes a collection and returns a new collection containing only the elements that satisfy a specific condition.

### The Imperative Way

Before functional patterns, we would create a "mutable bucket," loop through our data, and manually append items that meet our criteria.

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
var evenNumbers: [Int] = []

for number in numbers {
  if number % 2 == 0 {
    evenNumbers.append(number)
  }
}
// result: [2, 4, 6, 8, 10]
```

### The Functional Way

With `filter`, we get rid of the "how" (the loop and the manual appending) and focus on the "what" (the logic).

```swift
let evenNumbers = numbers.filter { $0 % 2 == 0 }
```

Just as `map` removes the logic, mutations, and iterations of how to create your new container, `filter` does the same thing, removing the logic of how to filter and instead focusing on what to filter.

## Reduce: The Assembler

`reduce` is used to combine all the items in a collection into a single value. This could be a sum, a string concatenation, or even a new dictionary.

### The Imperative Way

To sum an array, we usually create a variable at zero and add to it piece by piece.

```swift
let prices = [10.0, 20.0, 30.0]
var total: Double = 0

for price in prices {
  total += price
}
// total: 60.0
```

### The Functional Way

`reduce` takes two arguments: an initial value and a closure that defines how to combine the current running total with the next element.

```swift
let total = prices.reduce(0, { runningTotal, nextPrice in
  runningTotal + nextPrice
})

// Even shorter version using Swift's operator shorthand:
let total = prices.reduce(0, +)
```

## Composing the Trio: Map, Filter, and Reduce

The real power of functional programming in Swift is composition. Because these functions return new collections, you can chain them together to perform complex data processing in a single, readable pipeline.

The scenario: we have a list of orders. We want to find all orders over $50, apply a 10% tax to them, and then find the grand total.

### The All-in-One Pipeline

```swift
struct Order {
  let id: Int
  let amount: Double
}

let orders = [
  Order(id: 1, amount: 20.0),
  Order(id: 2, amount: 60.0),
  Order(id: 3, amount: 100.0)
]

let grandTotal = orders
  .filter { $0.amount > 50 } // Keep only expensive orders.
  .map { $0.amount * 1.10 }  // Add 10% tax.
  .reduce(0, +)              // Sum them up.

print(grandTotal) // 176.0
```

By chaining these functions together we have avoided creating multiple `var`s along the way. For instance, there is no need for an `ordersOver50` array. Or even an `ordersWithTax` array. Or a `totalValue` var to hold the accumulative total.

At each point we just describe what it is we want from the data.

## Wait... Can this be a Filter or Reduce?

We've all been there typing out yet another `for` loop. But before you hit enter, take a second to look at the "shape" of the code you're about to write. Just like we learned to spot a `map` by seeing a transformation, `filter` and `reduce` have their own distinct scents.

### The Filter Smell: The Selective Picker

If you find yourself writing a loop that contains a single `if` statement just to decide whether or not to append an item to a new array, you're writing a `filter`.

The sign: You have a bucket array, `var results = []`, and a loop that says: "If this specific thing is true, put it in the bucket."

The fix: Stop managing the bucket. Use `.filter { ... }` and just provide the truth logic.

### The Reduce Smell: The Master Assembler

If you see a standalone variable sitting right above a loop, like `var total = 0` or `var combinedString = ""`, and that variable gets updated every single time the loop ticks, you're looking at a `reduce`.

The sign: You are folding a whole list of items down into one single result. Whether you're summing prices, building a sentence, or even flattening nested data, if the list goes in and one single thing comes out, it's a `reduce`.

The fix: Ditch the mutable variable. Use `.reduce(initialValue) { ... }` to assemble your result in one clean shot.

Recognizing these patterns is like developing a superpower. Instead of reading how the computer is moving bits around, you start reading what your data is actually doing.

## Coming Up in Part 3: The Power of Reduce

While we used `reduce` here to sum numbers, it is secretly the most powerful function in the functional toolkit. In the next part, we will explore how `reduce` can actually build other functions like `map` and `filter`, and how to use it for complex tasks like partitioning data into groups, such as grouping objects by month.
