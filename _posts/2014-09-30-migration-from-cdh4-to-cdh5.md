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
At work, I had a task to migrate from CDH4 to CDH5.

<!--more-->

The process wasn't particularly difficult, but I encountered several issues:

1. Added Maven Enforcer to simplify finding dependencies with different versions.

2. As a consequence of moving to CDH5, we needed to upgrade to Protobuf 2.5 and Akka 2.3.

3. We had to split Hadoop/HBase dependencies. For example, to use TableMapReduceUtil (where the code should remain the same or almost the same), you need a dependency on hbase-server. So I needed to include it in dependencies (and exclude lots of underlying components).

4. For our CDH cluster, our admin uses LDAP users. So I needed to create HDFS folders and set HADOOP_USER_NAME in module configurations.

5. To run MapReduce tasks, the user also had to be changed. Using this approach from http://stackoverflow.com/questions/16108522/running-a-map-reduce-job-as-a-different-user, I added the following wrapper:

<pre class="toolbar:2 lang:scala decode:true">def runAs[T](f: => T): T = {
    val ugi = UserGroupInformation.createRemoteUser(hadoopEnvironment.user)
    ugi.doAs(new PrivilegedExceptionAction[T] {
      override def run(): T = {
        f
      }
    })
  }
</pre>

This required minimal modifications to existing code. Just wrap the old code like this:

<pre class="toolbar:2 lang:scala decode:true">runAs {
  <old code for Job creation>
}</pre>

6. I had some problems with Akka. I got an exception:

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

This took a couple of hours of my time to investigate. It was a binary incompatibility with a library compiled for Akka 2.2. But due to Enforcer, I didn't have any dependency on Akka 2.2, and I didn't have any exclusions for Akka.

The culprit turned out to be spray-can. It had a "provided" dependency. =(