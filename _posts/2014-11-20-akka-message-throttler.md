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

Today found in our sources solution for controlling messages rps from one actor to another. Solution was with with setting rps and Thread.sleep after each sending.
I know about maxima - use things which helps to solve the problem, but I do not think that much of threads here and there - good solution anyway. So I tried to write my own throttler, akka-style.
Yes, I found [experimental](http://doc.akka.io/docs/akka/snapshot/contrib/throttle.html) feature in AKKA 2.4. But not far ago I saw the situation when experimental feature does not work. So my bicycle is the best ))

~~~ scala
import akka.actor.{Actor, Props}

import scala.concurrent.duration.{FiniteDuration, _}
import scala.util.control.NonFatal

/**
 * Class to limit some action in time. First use - to send actor messages with given RPS.
 * Because we rely on scheduler, we process actions in batches with size actionsAmount, so
 * frequency is average
 * (see akka.scheduler.tick-duration as possible bottleneck )
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