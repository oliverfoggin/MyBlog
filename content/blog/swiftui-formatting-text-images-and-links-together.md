+++
title = "SwiftUI: Formatting text, images, and links together"
description = "How to combine SwiftUI text, links, and SF Symbol images while keeping link tinting consistent."
date = 2023-08-03

[taxonomies]
tags = ["swiftui", "swift", "ios"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/swiftui-formatting-text-images-and-links-together-82127a70b47b"
+++

## Introduction

I was given a task recently to create some text with links in it in our SwiftUI app. With one difference, the text had images in it and the images also took on the tint colour of the links. If you're looking for something similar you can find all the code at the bottom of this article.

## First attempt

My first start was to just use the markdown formatting of SwiftUI...

```swift
Text("Accept **[Terms and Conditions](https://oliverfoggin.com)** and **[Privacy Policy](https://oliverfoggin.com)**")
  .tint(.green)
```

A promising start. And then I added the images...

```swift
Text("Accept **[Terms and Conditions \(Image(systemName: "square.and.arrow.up"))](https://oliverfoggin.com)** and **[Privacy Policy \(Image(systemName: "square.and.arrow.up"))](https://oliverfoggin.com)**")
  .tint(.green)
```

This is where it got tricky as the images were not picking up the tint colour. And adding the `.foregroundColor` modifier to the `Image` gave an error.

## The solution

By wrapping the `Image` in `Text` we can finally append the color to make it look right...

```swift
Text("Accept **[Terms and Conditions \(Text(Image(systemName: "square.and.arrow.up")).foregroundColor(.green))](https://oliverfoggin.com)** and **[Privacy Policy \(Text(Image(systemName: "square.and.arrow.up")).foregroundColor(.green))](https://oliverfoggin.com)**")
  .tint(.green)
```

Except now the line of text is getting a bit out of hand. So extracting a couple of helpers made it much nicer to read.

```swift
Text("Accept \(terms) and \(privacy)")
  .tint(.green)

@ViewBuilder
private var linkIcon: Text {
  Text(Image(systemName: "square.and.arrow.up"))
    .foregroundColor(.green)
}

@ViewBuilder
private var privacy: Text {
  Text("**[Privacy Policy \(linkIcon)](https://oliverfoggin.com)**")
}

@ViewBuilder
private var terms: Text {
  Text("**[Terms and Conditions \(linkIcon)](https://oliverfoggin.com)**")
}
```

I was surprised at how straight forward it was to get this working as there seemed to be a million different ways recommended online.
