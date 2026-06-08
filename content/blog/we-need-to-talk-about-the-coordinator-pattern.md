+++
title = "We need to talk about The Coordinator Pattern"
description = "A look at when the Coordinator Pattern is useful, and why design patterns should emerge from the problem space rather than drive it."
date = 2023-08-03

[taxonomies]
tags = ["ios", "architecture", "coordinator-pattern", "design-patterns"]
categories = ["development"]

[extra]
source_url = "https://medium.com/@oliverfoggin/we-need-to-talk-about-the-coordinator-pattern-e3f8d6e251c9"
+++

## What is The Coordinator Pattern?

The Coordinator Pattern is a design pattern used in software engineering and has been around a long time and was made popular by a [blog post by Soroush Khanlou back in 2015](https://khanlou.com/2015/10/coordinators-redux/). Right about the time when the MVC pattern became a meme in itself and was given the name "Massive View Controller".

It is a design pattern used to decouple UIViewControllers from the navigation flows that they are a part of. And, at first glance, I can see how this might be useful.

## Why is it used?

The reasons for using this pattern that I have heard many times is that it means we no longer have to test navigation flow as part of the UIViewController. Which is true. Or that view controllers can be reused as part of different flows within an app. Which is also true. But with many of these I'm not sure that The Coordinator Pattern is any better a solution than the numerous other ways that these can be achieved.

Along with these reasons are the projects themselves that use the coordinator pattern. And in almost all of the cases where The Coordinator Pattern is used the flows within the app are fairly static. There are not many reused view controllers. The flows of the app are known at the time of development. And, when there is testing, the navigation flow is still tested statically. Only it is tested through the coordinators rather than through the view controllers.

Even worse, I often see The Coordinator Pattern being used where the view controllers are telling the coordinators specifically which view controllers to push onto the navigation stack. Which negates the point of using the pattern in the first place.

This leads me to ask the question, why is The Coordinator Pattern being used here? It seems like a lot of the benefits that are being described are not actually being put into use here. Or, if they are, the complexity that has come with adding coordinators has not removed complexity from elsewhere but only shifted that complexity to somewhere else. It seems to be used in a lot of places without really asking why it is used or what it's true benefit is.

Where is it that The Coordinator Pattern really shines?

## What is The Coordinator Pattern really good for?

Back in 2012, reasonably early in my career, before I really appreciated design patterns and really, before I was involved much in the iOS community, I was working on an app in an agency. This app was intended for people to go out into the field and use it as a sort of clipboard.

They would be able to search for existing clients, add new clients, enter their address, email, telephone number, search for products they were interested in, etc, etc... This seems like a fairly simple app to put together on the surface. I had a view controller for entering a phone number, one for searching for a postcode, one for searching for products, etc...

The problem I had with this app is that I did not know which flow the user would be starting and also the various flows that the user could pick from were all maintained online on a central server and so could change at any time. So, I had been backed into a corner where, not only could the view controllers not know what came before them or what came next but I, as the developer, also did not know what came before or after each view controller.

And so there was no where in the app that I could move this complexity to. [The answer I came up with was to use a "Flow Controller" as I called it at the time.](https://stackoverflow.com/a/11836763) It wasn't perfect and, as with every project I have worked on, there are things I would do differently looking back at it. But, I would argue that this is the one use case that The Coordinator Pattern is good for.

In this particular case I did not arbitrarily choose to decouple the navigation flow from the view controllers. I did not arbitrarily decide that I wanted to be able to reuse my view controllers. The problem space I was given had forced me into a position where that was my only choice. I could not have baked the navigation flow into the view controllers even if I had wanted to.

The architecture I used was not chosen out of a list of different architectural patterns, it was discovered by exploring the problem space and finding the best way of solving the problem that existed within the app I was writing.

## Why design patterns exist?

Design patterns are not prescribed ways of writing software, nor are they tools that can be used, they exist purely to describe commonly repeated solutions to various different problems. We don't use design patterns, we discover them through investigating the problem space that we are working in. And of course, those problem spaces are somewhat constrained by the platform we are working with too. Which is why MVC was the pattern of choice for many iOS apps in the early days.

Design patterns are also not a guarantee that your project will be good or well written. It is vital to constantly ask "Why?". Why are we writing the code this way? Is there a better way to write it? I see a lot of projects where it has been started out by stating "We are going to write this project using <insert design pattern here>." Almost like that means that all the problems have been solved and then very little thought is put into that architectural discussion after that.

An analogy I keep coming back to would be like a builder saying, "I'm going to build a new house and the tool I am going to use is a hammer." And then when they start having to cut down some trees for timber they take the hammer and start smashing through the tree with all their might because they said they were going to use a hammer and that's what they're going to use.

## Conclusion

So, I guess, this isn't really about just The Coordinator Pattern. As a pattern it has it's uses and in those cases (of which I argue there are very few) it really does work well. This post is more about why we have design patterns and what they mean to us as developers.

Don't start your next project with a view to using a certain architecture. Just, start the project and write your app, only by doing that will you start to discover sections of the app where repeated patterns exist that could possibly be extracted. You may even extract those patterns into something that looks a bit like a design pattern. That pattern might be The Coordinator Pattern, or it might be MVVM, or MVC, or MVP, or TEA, or POP, or any of a host of other ensign patterns that exist.

But don't stop there as you might find that the pattern you stumbled across is only suited for a particular part of your app. And that's fine too. There's no rule that states you must decide on a single pattern for your entire app. In my example there were other parts of the app that were well defined at the time of writing the app. The login screen for instance was well defined and didn't need to change and so I used plain old MVC for that part of the app.

Design Patterns are nice to read about and understand but don't let them be an alternative to thinking about how and why you are writing what you are writing. Only you know the problem space of the project that you're working in. And only you can understand whether a particular solution (or number of solutions) is right for it. Don't get stuck by thinking you have to stick with what has gone before just because it is written down in a blog or a book.

Anyway... have fun in writing your project. If you go wrong, learn from it. All that matters is that you keep thinking and keep having fun while doing it.
