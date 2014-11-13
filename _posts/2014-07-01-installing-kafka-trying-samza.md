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
1. Download kafka

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ mkdir kafka
$ cd kafka
$ wget http://apache.lauf-forum.at/kafka/0.8.1.1/kafka_2.10-0.8.1.1.tgz
$ tar xzvf kafka_2.10-0.8.1.1.tgz
$ cd kafka_2.10-0.8.1.1</pre>

<!--more-->

2. Update zookeeper properties

<pre class="toolbar:1 lang:default decode:true" title="vi config/server.properties">broker.id=&lt;unique id for node&gt;
zookeeper.connect=dev26i.vs.os.yandex.net:2181,dev27i.vs.os.yandex.net:2181,dev28i.vs.os.yandex.net:2181</pre>

3. Run it:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ bin/kafka-server-start.sh config/server.properties</pre>

Now [Samza][1]

1. Publish samza to local maven repo

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ git clone http://git-wip-us.apache.org/repos/asf/incubator-samza.git
$ cd incubator-samza
$ ./gradlew clean publishToMavenLocal</pre>

2. Will try hello-samza project. Using [this][2]

Had some problems here: version of samsa libs was just [changed][3] to 0.8.0

So I need to change:

<pre class="toolbar:1 nums:false lang:default decode:true" title="vi hello-samza/pom.xml">...
  &lt;artifactId&gt;samza-example-parent&lt;/artifactId&gt;
  &lt;version&gt;0.8.0&lt;/version&gt;
  &lt;packaging&gt;pom&lt;/packaging&gt;
...
        &lt;groupId&gt;samza&lt;/groupId&gt;
        &lt;artifactId&gt;samza-wikipedia&lt;/artifactId&gt;
        &lt;version&gt;0.8.0&lt;/version&gt;
...

and then all other "0.7.0" to "0.8.0-SNAPSHOT"

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

Added cloudera repo instead of apache because we use CDH5 YARN.

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

3. By the way on this node I do not have hadoop-client. See [here][4] how to install

4. Put tar.gz to HDFS

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ hadoop fs -put samza-job-package/target/samza-job-package-0.8.0-dist.tar.gz &lt;path on HDFS&gt;</pre>

5. And run the job

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ mkdir -p deploy/samza
$ tar -xvf ./samza-job-package/target/samza-job-package-0.8.0-dist.tar.gz -C deploy/samza
$ # set yarn path in deploy/samza/config/wikipedia-*.properties
$ deploy/samza/bin/run-class.sh org.apache.samza.job.JobRunner --config-factory org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-feed.properties</pre>

And check in [hadoop][5] (??? problems??)

and in Kafka

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.net:2181 --topic wikipedia-raw</pre>

&nbsp;

&#8230;

And here I got stuck for whole day. It does not work.. Found problems:

1. Tried &#8220;hand-made&#8221; feed with

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ bin/produce-wikipedia-raw-data.sh -b dev30i.vs.os.yandex.net:9093 -z dev27i.vs.os.yandex.net:2181</pre>

Does not work because of hardcoded paths to kafka. First I need to change paths to my installed kafka. But after that everything is ok. Kafka is working.. But what about YARN?

2. Checked YARN UI (on porn 8088). Found that samza job looks like:

STATE: ACCEPTED, FinalStatus: UNDEFINED, PROGRESS: stays as 0, Tracking UI: UNASSIGNED

Then after some time it becomes FAILED.

I went to Logs but cannot see them in UI. Got 404. I went on nodes which were marked as workers. And found that nodemanager is not running. Ups.. Restarted, rerun jobs and found again that nodemanager is not running.

I went to nodemanager logs. Found there:

<pre class="toolbar:2 lang:default decode:true">...
java.lang.IllegalArgumentException: Wrong FS: hdfs://var/log/hadoop-yarn/apps, expected: hdfs://dev27i.vs.os.yandex.net:8020
...</pre>

WTF??

After that I found that configuration mentioned in Cloudera installation manual was wrong:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">yarn.nodemanager.remote-app-log-dir: hdfs://var/log/hadoop-yarn/apps</pre>

But should be

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">yarn.nodemanager.remote-app-log-dir: /var/log/hadoop-yarn/apps</pre>

With cloudera setup nodemanager tries to start Application, failed with IllegalArgumentException on checkPath and shutdown after that!

Ok at the end Application successfully started.

3. But kafka consumer still shows nothing.

Just because in properties in package we uploaded to HDFS we did not change kafka broker urls. Ok, stop application in YARN, change properties, build package again, upload, and start application. And now everything is working! Hurrah!

&nbsp;

Then rest of tutorial

1. Start wikipedia edits job (according [this][6])

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ deploy/samza/bin/run-job.sh --config-factory=org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-parser.properties</pre>

and check

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.2181 --topic wikipedia-edits</pre>

2. Start statistics calculations

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ deploy/samza/bin/run-job.sh --config-factory=org.apache.samza.config.factories.PropertiesConfigFactory --config-path=file://$PWD/deploy/samza/config/wikipedia-stats.properties</pre>

and check

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ ~/kafka/kafka_2.10-0.8.1.1/bin/kafka-console-consumer.sh  --zookeeper dev27i.vs.os.yandex.2181 --topic wikipedia-stats</pre>

That&#8217;s all

 [1]: http://samza.incubator.apache.org/
 [2]: http://samza.incubator.apache.org/learn/tutorials/0.7.0/deploy-samza-job-from-hdfs.html
 [3]: https://issues.apache.org/jira/browse/SAMZA-304
 [4]: http://blog.forteamwork.ru/?p=23
 [5]: http://dev26i.vs.os.yandex.net:8088
 [6]: http://samza.incubator.apache.org/startup/hello-samza/0.7.0/