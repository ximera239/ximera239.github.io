---
title: Hadoop task debugging
status: published
author: Evgeny
layout: post
permalink: /2014-01-26-Hadoop-task-debugging/
---

Last week I was asked by collegue to help with retrieving logs from our hadoop cluster.
Because of my curiocity I looked into log files by myself. And found there strange (at least for me) thing. We got OOM on shuffle stage.

Because it is not clear what takes memory, I want to get heap dump on OOM. And this post about what I did for that.

I looked into [this](https://yhemanth.wordpress.com/2013/03/28/taking-memory-dumps-of-hadoop-tasks/) post with link to
[this](http://mail-archives.apache.org/mod_mbox/hadoop-user/201303.mbox/%3C14D8D8E3-9341-4F90-9AA5-DF466AD0E5B9%40yahoo-inc.com%3E) post.
It is written for MR1, so some modifications should be done for YARN.

 * For user which run this job I added two folders on HDFS:

~~~
$ hadoop fs -ls /user/test
...
drwxr-xr-x   - test test          0 2015-01-26 12:40 /user/test/hprof
drwxr-xr-x   - test test          0 2015-01-26 12:40 /user/test/scripts
...
~~~

 * I put script upload_dump.sh to hdfs

~~~
$ hadoop fs -cat /user/test/scripts/upload_dump.sh
#!/bin/sh
hadoop dfs -put myheapdump.hprof /user/test/hprof/${PWD//\//_}.hprof
~~~

 * I added additional params to job configuration (I used scala - ${HDFS.username} is my environment hadoop user name):

~~~
mapreduce.reduce.java.opts -> """-XX:+HeapDumpOnOutOfMemoryError
                              -XX:HeapDumpPath=./myheapdump.hprof
                              -XX:OnOutOfMemoryError=./upload_dump.sh"""
mapreduce.job.cache.files -> s"hdfs:///user/${HDFS.username}/scripts/upload_dump.sh#upload_dump.sh"
~~~

Hope, this will help me to get dump..