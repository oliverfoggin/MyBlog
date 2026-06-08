+++
title = "Swift Syntax Highlighting"
description = "A quick check that Swift code blocks render with syntax highlighting."
date = 2026-06-08

[taxonomies]
tags = ["swift", "zola", "syntax-highlighting"]
categories = ["development"]
+++

This post checks that fenced Swift snippets render with syntax highlighting.

```swift
import Foundation

struct Post: Identifiable, Sendable {
  let id: UUID
  var title: String
  var publishedAt: Date
}

extension Post {
  var slug: String {
    title
      .lowercased()
      .split(whereSeparator: { !$0.isLetter && !$0.isNumber })
      .joined(separator: "-")
  }
}
```
