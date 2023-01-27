---
layout: post
title:  "You're never a sole developer"
date:   2023-01-10 11:57:04 +0000
categories: process, flow, development
---

As a developer, it's easy to fall into the trap of thinking that you're working alone on a project - especially when you're working alone on a project.

However, the reality is that you're always part of a team – a team that exists throughout space and time. You're constantly working with past versions of yourself and future versions of yourself, and it's important to keep this in mind when you're writing code.

When you're in a deep flow state and coding is coming easily - pouring out of your hands into the IDE like a montage scene form a 90's hacker movie. When things like tests, documentation and comments aren't part of your process, they become a hinderance, at the time they seem like a _waste_ of time.

Picture the scene: You've just released a new feature on your application and it's a huge success. A month later, a user requests a tiny change to that feature. You haven't written tests, documentation, or comments, some parts of the code have you in complete confusion, "who the fuck wrote this? what the fuck does this do?". This can make it difficult or more interestingly _unenjoyable_ to make the change that the user has requested. Which slows you down. It leads to delays. You put off the change. It goes to the bottom of the to do list. It sits in your mind, nagging at you.

It's easy to avoid this: Always work as if you're part of a team, even if you're on your own. Write unit tests, integration tests, and documentation where things get tricky, or you haven't done something before so it's not quite retained yet, or there are things that you'll need to remember. Write clean, dry code, use an infrastructure-as-code (IaC) tool, have the tests run in a continuous integration/continuous delivery (CI/CD) pipeline.

While this may slow you down initially - adding these things to your process, your flow state, it will save you time and frustration in the long run. When a change request comes in or it's time to experiment, you'll know that you haven't done any damage because the tests will fail. You'll know that the infrastructure is sound because it's all in IaC, and you'll know what that strange looking bit of code is doing because you've documented it.

In short, you're never a "sole developer", even if you're working on your own – you're always part of a team that exists throughout space and time. You're a team made of you from 1 month ago, fresh and excited for the day, and another you from 3 months ago, burnt out and tired just trying to get things to work.

By writing tests, documentation, and comments, you're being an awesome team mate to everyone – especially yourself.
