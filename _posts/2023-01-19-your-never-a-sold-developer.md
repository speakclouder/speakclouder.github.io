---
layout: post
title:  "You're never a sole developer"
date:   2023-01-10 11:57:04 +0000
categories: process, flow, development
---

As a developer, it's easy to fall into the trap of thinking that you're working alone on a project - especially when you're working alone on a project. 

However, the reality is that you're always part of a team – a team that exists throughout space and time. You're constantly working with past versions of yourself and future versions of yourself, and it's important to keep this in mind when you're writing code.

When you're in a deep flow state and coding is coming easily, it's easy to forget about the need for documentation, comments, and tests. However, these things are vital. Get them into your default workflow - they should be part of your flow state. A must.

Picture the scene: You've just released a new feature on your application and it's a huge success. A month later, a user requests a tiny change to that feature. You haven't written tests, documentation, or comments, some parts of the code have you in complete confusing - "who the fuck wrote this?". This can make it difficult to make the change that the user has requested and can even lead to delays in getting the feature live.

To avoid this, it's important to always work as if you're part of a team, even if you're on your own. Write unit tests, integration tests, and documentation where things get tricky or there are things that you'll need to remember. Write clean, dry code, use an infrastructure as code (IaC) tool, have the tests run in a continuous integration/continuous delivery (CI/CD) pipeline.

While this may slow you down initially, it will save you time and frustration in the long run. When a change request comes in or it's time to experiment, you'll know that you haven't done any damage because the tests will fail. You'll also know that the infrastructure is sound because it's all in IaC, and you'll know what that strange bit of code is doing because you've documented it.

In short, you're never a "sole developer" – you're always part of a team that exists throughout space and time. You're a team made of you from 1 month ago fresh and excited for the day, and you from 3 months ago, burnt out just trying to get things to work.

By writing tests, documentation, and comments, you're being an awesome team mate to everyone – especially yourself.
