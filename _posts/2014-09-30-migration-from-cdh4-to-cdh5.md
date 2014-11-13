---
title: Migration from CDH4 to CDH5
author: Evgeny
layout: post
permalink: /2014-09-30-migration-from-cdh4-to-cdh5/
categories:
  - Uncategorized
tags:
  - CDH4
  - CDH5
---
At work I had a task to migrate from CDH4 to CDH5

<!--more-->

Not so hard but several issues I had:

1. Added maven enforcer to simplify finding of different version dependencies.

2. As subsequence of moving to CDH5 we need to mode to protobuf 2.5 and to akka 2.3

3. Splitting hadoop/hbase dependencies. For example, to use TableMapReduceUtil (code should stay the same or almost the same) you need dependency to hbase-server. So I need to include it to dependencies (and exclude lots of underlying staff)

2. For CDH our admin use LDAP users. So I need to create HDFS folders and set HADOOP\_USER\_NAME in modules configurations

4. To run MR tasks user also should be changes. Using this  http://stackoverflow.com/questions/16108522/running-a-map-reduce-job-as-a-different-user I added such wrapper:

<pre class="toolbar:2 lang:scala decode:true">def runAs[T](f: =&gt; T): T = {
    val ugi = UserGroupInformation.createRemoteUser(hadoopEnvironment.user)
    ugi.doAs(new PrivilegedExceptionAction[T] {
      override def run(): T = {
        f
      }
    })
  }
</pre>

So minimum modification in existing code is required. Just wrap:

<pre class="toolbar:2 lang:scala decode:true">runAs {
&lt;old code for Job creating&gt;
}</pre>

5. Had some problems with akka. I got an exception:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true ">java.lang.AbstractMethodError
	at akka.actor.ActorCell.create(ActorCell.scala:580)
	at akka.actor.ActorCell.invokeAll$1(ActorCell.scala:456)
	at akka.actor.ActorCell.systemInvoke(ActorCell.scala:478)
	at akka.dispatch.Mailbox.processAllSystemMessages(Mailbox.scala:263)
	at akka.dispatch.Mailbox.run(Mailbox.scala:219)
	at akka.dispatch.ForkJoinExecutorConfigurator$AkkaForkJoinTask.exec(AbstractDispatcher.scala:393)
	at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)
</pre>

This takes couple hours of my time to investigate. It was binary incompatibility of some lib compiled for akka 2.2. But &#8211; because of enforser I do not have any dependency of akka 2.2. And I do not have any excludes of akka.

And.. it was spray-can. It has &#8220;provided&#8221; dependency =(

&nbsp;

&nbsp;

&nbsp;