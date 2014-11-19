---
title: Hack Sequel in Berlin
author: Evgeny
layout: post
permalink: /2014-11-19-hach-sequel-in-berlin/
categories:
  - Uncategorized

tags:
  - stories

---

Attended [hackathon](http://www.meetup.com/Scala-Berlin-Brandenburg/events/213681812/) in Berlin. Was quite nice and new experience for me.

The idea was to create something in one day in team. My goal was to meet new people, to learn something new.
Our team was: two guys from Typesafe, Two guys from ImmobilienScout24 and myself. Project was about to visualize akka-tree.

In general - we collect events with aspectJ (for the project - actor creation, actor termination, and queue size), send it to Kafka (of cause another solution is possible, but I wanted to learn Kafka :) ), than reading from Kafka, wrapping to akka-stream, and send it to client with [SSD](http://en.wikipedia.org/wiki/Server-sent_events)
On client side it is a [D3](http://d3js.org/) managed page which show growing akka system in real time.

We had several problems:
1. Looks like akka streams lose messages
2. Client works only in Safary in OS X Yosemite

Project on [github](https://github.com/nraychaudhuri/akka-tree/)

