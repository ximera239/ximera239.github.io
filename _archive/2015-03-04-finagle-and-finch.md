---
title: Finagle and Finch
status: published
author: Evgeny
layout: post
permalink: /2015-03-04-finagle-and-finch/
---

I would like to learn more about different approaches to building RESTful services.

I heard about [Finagle](http://twitter.github.io/finagle/) some time ago, but hadn't found the time to look into it until now. Today I came across a [blog post](http://habrahabr.ru/post/250591/) on [HABRAHABR](habrahabr.ru) about building REST APIs on top of Finagle using [Finch](https://github.com/finagle/finch), a brand new extension to Finagle designed for building RESTful services.

After reading it, I got the impression that this is very similar to spray-route which I currently use. The main differences seem to be in naming and background architecture. Spray is built on top of AKKA, while Finch is built on top of Finagle. It would be interesting to compare implementations of the same service using both approaches...

By the way, I found a [post](http://blog.samibadawi.com/2013/04/akka-vs-finagle-vs-storm.html) with a comparison of AKKA, Storm, and Finagle. Looks interesting.