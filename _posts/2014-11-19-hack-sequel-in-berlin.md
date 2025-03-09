---
title: Hack Sequel in Berlin
author: Evgeny
layout: post
permalink: /2014-11-19-hack-sequel-in-berlin/
categories:
  - Uncategorized

tags:
  - stories

---

I attended a [hackathon](http://www.meetup.com/Scala-Berlin-Brandenburg/events/213681812/) in Berlin. It was a new experience for me.

The idea was to create something in one day as a team. My goal was to meet new people and learn something new.
Our team consisted of: two guys from Typesafe, two guys from ImmobilienScout24, and myself. The project was about visualizing the actor model as a tree.

In general, we collected events with AspectJ (project scopes were actor creation, actor termination, and queue size), sent them to Kafka (of course another solution would have been possible, but I wanted to learn Kafka :)), then read from Kafka, wrapped it in akka-stream, and sent it to the client with [SSE](http://en.wikipedia.org/wiki/Server-sent_events).
On the client side, it was a [D3](http://d3js.org/)-managed page which showed the growing Akka system in real time.

We had several problems:
1. It looked like Akka streams were losing messages
2. The client only worked in Safari on OS X Yosemite

The project is on [GitHub](https://github.com/nraychaudhuri/akka-tree/)