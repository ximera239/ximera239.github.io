---
title: Installing Kafka, trying Samza
author: Evgeny
layout: post
permalink: /2014-07-01-installing-kafka-trying-samza/
categories:
  - Big Data
tags:
  - CDH5
  - Installation manual
  - Kafka
  - Samza
  - YARN
  - ZooKeeper
---
1. Download Kafka

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ mkdir kafka
$ cd kafka
$ wget http://apache.lauf-forum.at/kafka/0.8.1.1/kafka_2.10-0.8.1.1.tgz
$ tar xzvf kafka_2.10-0.8.1.1.tgz
$ cd kafka_2.10-0.8.1.1</pre>

<!--more-->

2. Update ZooKeeper properties

<pre class="toolbar:1 lang:default decode:true" title="vi config/server.properties">broker.id=<unique id for node>
zookeeper.connect=dev26i.vs.os.yandex.net:2181,dev27i.vs.os.yandex.net:2181,dev28i.vs.os.yandex.net:2181</pre>

3. Run it:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ bin/kafka-server-start.sh config/server.properties</pre>

Now let's set up [Samza][1]

1. Publish Samza to local Maven repository

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ git clone http://git-wip-us.apache.org/repos/asf/incubator-samza.git
$ cd incubator-samza
$ ./gradlew clean publishToMavenLocal</pre>

2. Try the hello-samza project using [this guide][2]

I encountered some problems here: the version of Samza libs had just [changed][3] to 0.8.0

So I needed to make these changes:

<pre class="toolbar:1 nums:false lang:default decode:true" title="vi hello-samza/pom.xml">...
  &lt;artifactId&gt;samza-example-parent&lt;/artifactId&gt;
  &lt;version&gt;0.8.0&lt;/version&gt;
  &lt;packaging&gt;pom&lt;/packaging&gt;
...
        &lt;groupId&gt;samza&lt;/groupId&gt;
        &lt;artifactId&gt;samza-wikipedia&lt;/artifactId&gt;
        &lt;version&gt;0.8.0&lt;/version&gt;
...

and then change all other "0.7.0" to "0.8.0-SNAPSHOT"

...
    &lt;repository&gt;
      &lt;id&gt;local_repo&lt;/id&gt;
      &lt;url&gt;file://${user.home}/.m2/repository&lt;/url&gt;
    &lt;/repository&gt;
    &lt;repository&gt;
      &lt;id&gt;cloudera&lt;/id&gt;
      &lt;url&gt;https://repository.cloudera.com/artifactory/cloudera-repos/&lt;/url&gt;
    &lt;/repository&gt;
..</pre>

I added the Cloudera repository instead of Apache because we use CDH5 YARN.

<pre class="toolbar:1 nums:false lang:default decode:true" title="vi hello-samza/samza-job-package/pom.xml">&lt;parent&gt;
    &lt;groupId&gt;samza&lt;/groupId&gt;
    &lt;artifactId&gt;samza-example-parent&lt;/artifactId&gt;
    &lt;version&gt;0.8.0&lt;/version&gt;
  &lt;/parent&gt;
...
    &lt;dependency&gt;
      &lt;groupId&gt;org.apache.hadoop&lt;/groupId&gt;
      &lt;artifactId&gt;hadoop-hdfs&lt;/artifactId&gt;
      &lt;version&gt;2.3.0-cdh5.0.2&lt;/version&gt;
    &lt;/dependency&gt;</pre>

<pre class="toolbar:1 nums:false lang:default decode:true" title="vi hello-samza/samza-wikipedia/pom.xml">&lt;parent&gt;
    &lt;groupId&gt;samza&lt;/groupId&gt;
    &lt;artifactId&gt;samza-example-parent&lt;/artifactId&gt;
    &lt;version&gt;0.8.0&lt;/version&gt;
  &lt;/parent&gt;</pre>

And then:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">hello-samza$ mvn clean package</pre>

3. Note that on this node I did not have hadoop-client. See [here][4] for installation instructions.

4. Put the tar.gz to HDFS

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ hadoop fs -put samza-job-package/target/samza-job-package-0.8.0-dist.tar.gz <path on HDFS></pre>

5. Run the job

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ mkdir -p deploy/samza
$ tar -xvf ./samza-job-package/target/samza-job-package-0.8.0-dist.tar.gz -C deploy/samza
$ # set yarn path in deploy/samza/config/wikipedia-*.properties
$ deploy/samza/bin/run-class.sh org.apache.samza.job.JobRunner --config-factory org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-feed.properties</pre>

Check in [Hadoop UI][5] and in Kafka:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.net:2181 --topic wikipedia-raw</pre>

&nbsp;

I was stuck here for a whole day. It didn't work. Here are the problems I found:

1. I tried the "hand-made" feed with:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ bin/produce-wikipedia-raw-data.sh -b dev30i.vs.os.yandex.net:9093 -z dev27i.vs.os.yandex.net:2181</pre>

It didn't work because of hardcoded paths to Kafka. First, I needed to change paths to my installed Kafka. After that, everything was OK. Kafka was working, but what about YARN?

2. I checked the YARN UI (on port 8088). I found that the Samza job status was:

STATE: ACCEPTED, FinalStatus: UNDEFINED, PROGRESS: stays as 0, Tracking UI: UNASSIGNED

Then after some time it became FAILED.

I tried to check the logs in the UI but got a 404 error. I went to the nodes that were marked as workers and found that the NodeManager was not running. I restarted it, reran the jobs, and found that the NodeManager stopped again.

In the NodeManager logs, I found:

<pre class="toolbar:2 lang:default decode:true">...
java.lang.IllegalArgumentException: Wrong FS: hdfs://var/log/hadoop-yarn/apps, expected: hdfs://dev27i.vs.os.yandex.net:8020
...</pre>

After investigating, I found that the configuration mentioned in the Cloudera installation manual was incorrect:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">yarn.nodemanager.remote-app-log-dir: hdfs://var/log/hadoop-yarn/apps</pre>

It should be:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">yarn.nodemanager.remote-app-log-dir: /var/log/hadoop-yarn/apps</pre>

With the Cloudera setup, the NodeManager tried to start the Application, failed with an IllegalArgumentException on checkPath, and shut down!

After fixing this, the Application successfully started.

3. However, the Kafka consumer still showed nothing.

This was because in the properties file in the package we uploaded to HDFS, we hadn't changed the Kafka broker URLs. I stopped the application in YARN, changed the properties, built the package again, uploaded it, and started the application. Now everything worked! Success!

&nbsp;

Then I followed the rest of the tutorial:

1. Started the Wikipedia edits job (according to [this guide][6]):

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ deploy/samza/bin/run-job.sh --config-factory=org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-parser.properties</pre>

and checked:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.2181 --topic wikipedia-edits</pre>

2. Started statistics calculations:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ deploy/samza/bin/run-job.sh --config-factory=org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-stats.properties</pre>

and checked:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.2181 --topic wikipedia-stats</pre>

That's all!

 [1]: http://samza.incubator.apache.org/
 [2]: http://samza.incubator.apache.org/learn/tutorials/0.7.0/deploy-samza-job-from-hdfs.html
 [3]: https://issues.apache.org/jira/browse/SAMZA-304
 [4]: http://blog.forteamwork.ru/?p=23
 [5]: http://dev26i.vs.os.yandex.net:8088
 [6]: http://samza.incubator.apache.org/startup/hello-samza/0.7.0/