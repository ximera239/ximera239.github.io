---
title: Hadoop task debugging
status: published
author: Evgeny
layout: post
permalink: /2014-01-26-Hadoop-task-debugging/
---

Last week a colleague asked me to help retrieve logs from our Hadoop cluster.
Out of curiosity, I looked into the log files myself and found something strange (at least to me). We got an OOM (Out of Memory) error during the shuffle stage.

Since it's not clear what's consuming the memory, I wanted to get a heap dump when OOM occurs. This post describes what I did to accomplish that.

I referenced [this](https://yhemanth.wordpress.com/2013/03/28/taking-memory-dumps-of-hadoop-tasks/) post which links to
[this](http://mail-archives.apache.org/mod_mbox/hadoop-user/201303.mbox/%3C14D8D8E3-9341-4F90-9AA5-DF466AD0E5B9%40yahoo-inc.com%3E) post.
Since these instructions were written for MR1, some modifications needed to be made for YARN.

 * For the user running this job, I added two folders on HDFS:

~~~
$ hadoop fs -ls /user/test
...
drwxr-xr-x   - test test          0 2015-01-26 12:40 /user/test/hprof
drwxr-xr-x   - test test          0 2015-01-26 12:40 /user/test/scripts
...
~~~

 * I put the script upload_dump.sh to HDFS:

~~~
$ hadoop fs -cat /user/test/scripts/upload_dump.sh
#!/bin/sh
hadoop dfs -put myheapdump.hprof /user/test/hprof/${PWD//\//_}.hprof
~~~

 * I added additional parameters to the job configuration (I used Scala - ${HDFS.username} is my environment Hadoop user name):

~~~
mapreduce.reduce.java.opts -> """-XX:+HeapDumpOnOutOfMemoryError
                              -XX:HeapDumpPath=./myheapdump.hprof
                              -XX:OnOutOfMemoryError=./upload_dump.sh"""
mapreduce.job.cache.files -> s"hdfs:///user/${HDFS.username}/scripts/upload_dump.sh#upload_dump.sh"
~~~

I hope this will help me get the dump.


Part 2.
The task runs regularly, and it's not clear if it was executed after my changes to enable dumping. Meanwhile, I found [this](http://jason4zhu.blogspot.ru/2014/11/shuffle-error-by-java-lang-out-of-memory-error-java-heap-space.html)
post about a similar error.