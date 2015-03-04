---
title: Finagle and Finch
status: published
author: Evgeny
layout: post
permalink: /2015-03-04-finagle-and-finch/
---

I would like to know a bit more about other ways to build RESTful services.

I heard about [Finagle](http://twitter.github.io/finagle/) some time ago, but before I did not
find some time to look into. Today I found [blog-post](http://habrahabr.ru/post/250591/) on [HABRAHABR](habrahabr.ru) about REST-api
building on top of Finagle with usage of [Finch](https://github.com/finagle/finch), brand new extension to Finagle for
building RESTful services.

After reading I got an impression that this is very similar to spray-route which I use. Only difference in naming. And background.
Spray is build on top of AKKA. Finch is build on top of Finagle. Suppose that it should be interesting to compare
implementations of same service...

By the way, I found [post](http://blog.samibadawi.com/2013/04/akka-vs-finagle-vs-storm.html) with kind of comparision
of AKKA, Storm and Finagle. Looks nice.
