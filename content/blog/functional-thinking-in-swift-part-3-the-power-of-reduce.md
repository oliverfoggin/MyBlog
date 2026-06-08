+++
title = "Functional Thinking in Swift: Part 3 - The power of Reduce"
description = "How Swift's reduce can derive map and filter, and how to use it to build reusable grouping tools."
date = 2026-03-07

[taxonomies]
tags = ["ios", "swift", "functional-programming"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/functional-thinking-in-swift-part-3-the-power-of-reduce-22aaea5b1716"
+++

In [Part 2](@/blog/functional-thinking-in-swift-part-2-filter-reduce.md) we looked at `filter` and `reduce` and how we can use them to shape the data that we start with. Removing elements with `filter` or even combining them like adding numbers with `reduce`.

In this part we will show how `reduce` is the powerhouse of the functional tools available. How it can be used to perform the same tasks that `filter` and `map` perform, but can also be used to create some really powerful ways of manipulating data.

## Map with reduce

To show how `reduce` is really the primitive function that can be used to derive others, let's look at how we might `map` an array and convert that to `reduce`.

```swift
let numbers = [1, 2, 3, 4]

// The Map version
let doubled = numbers.map { $0 * 2 }

// The Reduce version
let doubledWithReduce = numbers.reduce(into: []) { acc, number in
  acc.append(number * 2)
}
```

Here we can see that `map` is just a specific implementation of `reduce` where the accumulator is a new array and each operation appends into that new array.

## Filter with reduce

Just as with `map`, we can also perform a `filter` with `reduce`.

```swift
let numbers = [1, 2, 3, 4]

// The Filter version
let evens = numbers.filter { $0.isMultiple(of: 2) }

// The Reduce version
let evensWithReduce = numbers.reduce(into: []) { acc, number in
  if number.isMultiple(of: 2) {
    acc.append(number)
  }
}
```

You can see a similar pattern here that we used for `map`. We start from an empty array, but this time we only append if the number matches our condition.

## Reduce() and Reduce(into:)

In both these examples we use `reduce(into: [])`. This gives us an `inout` value that we can mutate in the closure and can be more performant when we're returning a collection.

The alternative is to use the version `reduce([])`, in which we still provide an initial value, but this is not mutable in the closure and we must return the new value. This is better for use when we're reducing into a single value like summing an array of ints.

## Partitioning by value

Here we've seen `reduce` not just outputting a single value but an array. We can go further by really pushing the power of `reduce`.

If we have an array of dates, we might want to split those dates into groups of the same calendar month. At first this seems like it might be easier to start building up from scratch the imperative way. Start with an empty array, create a new array to build up the first month, then append to the parent array when reaching a new month, etc.

But we can reduce (pun intended) this down in complexity and solve the problem once so that we can partition any sorted array into like groups.

## Thinking in data pipelines

When using functional ideas I often find it easier to think about data pipelines. That is, what is it that we want to get out of the data at each point.

For this problem we start with a sorted array of dates:

```swift
[date1, date2, date3, ..., dateN]
```

And out of it we want a nested array of dates where each subarray contains dates from the same calendar month.

```swift
[[date1, date2], [date3, date4, date5], ..., [..., dateN]]
```

As we work along the input we're going to have to ask ourselves what are the states we can be in? In this case, there are three possible states.

- The output array is empty. We haven't started yet.
- The last sub-array of the output array contains dates in the same month as the next date.
- The last sub-array of the output array contains dates in a different month as the next date.

In the above cases we can deal with each by:

- Just append a new array with the next date in it.
- Append the next date into the last sub-array.
- Just append a new array with the next date in it.

We can see that 1 and 3 require the same work, so we can combine these. Once we have these written down we can start to implement the grouping function.

```swift
let sortedDates: [Date] = ...

// We know we need a nested array.
let groupedByMonth: [[Date]] = sortedDates.reduce(into: []) { acc, next in
  // First deal with case 2.
  if let lastDate = acc.last?.last,
     Calendar.current.isDate(next, equalTo: lastDate, toGranularity: .month)
  {
    acc[acc.count - 1].append(next)
  } else {
    // Everything else goes in here.
    acc.append([next])
  }
}
```

This is great, but what if we want to group by dates in the same day, or year? Or what if we want to group by something else completely? Perhaps we want to group students by grade boundaries?

## Extracting the logic

When working with functional patterns we often find that once we've written something like the above we can dissect it to remove the specifics about that scenario but still keep the logic around what it's doing.

In this case, the specific logic is the check that the next date is in the same month as the last. Everything else is just the work to build the chunked array.

We can extract that and allow the developer to provide that logic for us. Let's create a new `group` function that does just this by providing a closure. The only thing we need to know is, given two elements, should they be in the same group or different. So we need two elements in and a bool out.

```swift
extension Array {
  // This is our new function with our closure.
  func group(by matches: (Element, Element) -> Bool) -> [[Element]] {
    // Here we can use the same reduce but generalise it with the closure.
    reduce(into: []) { acc, next in
      if let prev = acc.last?.last,
         matches(prev, next) {
        acc[acc.count - 1].append(next)
      } else {
        acc.append([next])
      }
    }
  }
}

let sortedDates = [...]

// Now we can group by month.
let groupedByMonth = sortedDates.group {
  Calendar.current.isDate($0, equalTo: $1, toGranularity: .month)
}

// Or we can group by day.
let groupedByDay = sortedDates.group {
  Calendar.current.isDate($0, equalTo: $1, toGranularity: .day)
}

// Or we can sort by something else entirely.
let groupedByGradeBoundary = sortedStudents.group {
  $0.grade == $1.grade
}
```

By doing this we have created a reusable chunking algorithm that we can take to any project and reuse any time we need to chunk our data this way.

We could even chain this together to group by year then month.

```swift
let monthYearGroups = sortedDates
  .group { last, next in // (Date, Date) -> Bool
    Calendar.current.isDate(next, equalTo: last, toGranularity: .month)
  }
  .group { last, next in // ([Date], [Date]) -> Bool
    // This algorithm ensures that both values have at least one element,
    // so we can safely force unwrap the first.
    Calendar.current.isDate(next.first!, equalTo: last.first!, toGranularity: .year)
  }
```

This small reusable function that we have created has encapsulated the work of grouping arrays into matching groups and allowed us to put the hard work in up front of understanding the problem and finding a solution.

## When to build functional tools

Building functional tools like this builds up a toolkit of functions that can be used across multiple projects and each one is solved, tested, verified, and ready to be plugged in to your data.

The trigger for building a function like this is boilerplate friction. If you find yourself writing the scaffolding, the empty arrays, the loops, the index management, more than the actual business logic, you've found a candidate for a functional tool. Move the how into an extension so your main code can stay focused on the what.
