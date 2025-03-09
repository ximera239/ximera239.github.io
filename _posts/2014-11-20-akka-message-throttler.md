---
title: Akka message throttler
author: Evgeny
layout: post
permalink: /2014-11-20-akka-message-throttler/
categories:
  - Uncategorized

tags:
  - scala

---

Today, I found a solution in our codebase for controlling Akka messaging RPS (requests per second). The solution involved setting RPS and using Thread.sleep after each message sent.
I know about the maxim "use things that help solve the problem," but I don't think that many threads are a good solution anyway. So I tried to write my own throttler, Akka-style.
Yes, I found an [experimental](http://doc.akka.io/docs/akka/snapshot/contrib/throttle.html) feature in AKKA 2.4. Recently, I faced a situation when an experimental feature didn't work. So my own solution is the best ))

~~~ scala
import akka.actor.{Actor, Props}

import scala.concurrent.duration.{FiniteDuration, _}
import scala.util.control.NonFatal

/**
 * Class to limit some action in time. First use - to send actor messages with given RPS.
 * Because we rely on scheduler, we process actions in batches with size actionsAmount, so
 * frequency is average
 * (see akka.scheduler.tick-duration as possible bottleneck)
 *
 * actionsAmount - maximum amount of calls to throttledTransformation per tick (which is 1 second by default)
 * i1 - generates parameters for throttledTransformation
 * finalAction is called at the end in case if no Exceptions occur
 * onError called in case of NonFatal exception during throttledTransformation. Process is stopped and finalAction
 *    will not be called
 * User: Evgeny Zhoga
 * Date: 20.11.14
 */
class Throttler[A](actionsAmount: Int,
                   i1: Iterator[A],
                   throttledTransformation: A => Unit,
                   finalAction: => Unit,
                   onError: Throwable => Unit,
                   tick: FiniteDuration
                    ) extends Actor {
  var start: Long = 0
  self ! "throttle"

  override def receive: Receive = {
    case "throttle" =>
      val diff = System.currentTimeMillis() - start
      if (diff >= tick.toMillis) self ! "process"
      else context.system.scheduler.scheduleOnce((tick.toMillis - diff) milliseconds, self, "process")(context.dispatcher, self)
    case "process" =>
      start = System.currentTimeMillis()
      try {
        val i = Iterator.fill(actionsAmount)(next()).flatten
        if (i.isEmpty) self ! "finalize"
        else {
          i.foreach(throttledTransformation)
          self ! "throttle"
        }
      } catch {
        case NonFatal(e) =>
          self ! "error"
      }
      start = System.currentTimeMillis()
    case "finalize" =>
      finalAction
      context.stop(self)
    case "error" =>
      onError
      context.stop(self)
  }

  def next(): Option[A] = {
    if (i1.hasNext) Some(i1.next())
    else None
  }
}

object Throttler {
  def props[A](actionsAmount: Int,
               i1: Iterator[A],
               throttledTransformation: A => Unit,
               finalAction: => Unit,
               tick: FiniteDuration = 1.seconds) =
    Props(new Throttler(actionsAmount,
      i1,
      throttledTransformation,
      finalAction,
      (e) => finalAction,
      tick))
  def props[A](actionsAmount: Int,
               i1: Iterator[A],
               throttledTransformation: A => Unit,
               finalAction: => Unit,
               onError: Throwable => Unit,
               tick: FiniteDuration = 1.seconds) =
    Props(new Throttler(actionsAmount,
      i1,
      throttledTransformation,
      finalAction,
      onError,
      tick))
}
~~~