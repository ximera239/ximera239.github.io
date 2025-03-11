---
title: Installing CDH5. HBase
author: Evgeny
layout: post
permalink: /2014-06-24-installing-cdh5-hbase/
categories:
  - Big Data
tags:
  - CDH5
  - HBase
  - Installation manual
  - ZooKeeper
---
Following [this guide][1]

1. Install the HBase package

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true pre codeblock">$ sudo apt-get install hbase</pre>

<!--more-->

2. HBase uses lots of files and the default limit of 1024 is too low. Update <samp class="ph codeph">/etc/security/limits.conf</samp>

<pre class="toolbar:1 lang:default decode:true" title="sudo vi /etc/security/limits.conf">hdfs  -       nofile  32768
hbase -       nofile  32768</pre>

And apply changes in /etc/pam.d/common-session

<pre class="toolbar:1 lang:default highlight:0 decode:true" title="sudo vi /etc/pam.d/common-session">session required  pam_limits.so</pre>

3. Check (and set if not already configured) the value of dfs.datanode.max.xcievers (upper bound on the number of files that a datanode can serve at any one time) in hdfs-site.xml

<pre class="toolbar:1 lang:default decode:true" title="sudo vi hdfs-site.xml">&lt;property&gt;
    &lt;name&gt;dfs.datanode.max.xcievers&lt;/name&gt;
    &lt;value&gt;4096&lt;/value&gt;
  &lt;/property&gt;
</pre>

Sync configurations and restart HDFS.

Roles:
- Node2, Node3: HBase Region servers
- Node1: HBase master

4. Install hbase-master on node1

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo apt-get install hbase-master</pre>

5. Configure hbase-site.xml and sync

<pre class="toolbar:1 lang:default decode:true" title="sudo vi /etc/hbase/conf/hbase-site.xml">&lt;configuration&gt;
&lt;property&gt;
  &lt;name&gt;hbase.cluster.distributed&lt;/name&gt;
  &lt;value&gt;true&lt;/value&gt;
&lt;/property&gt;
&lt;property&gt;
  &lt;name&gt;hbase.rootdir&lt;/name&gt;
  &lt;value&gt;hdfs://dev27i.vs.os.yandex.net:8020/hbase&lt;/value&gt;
&lt;/property&gt;
&lt;property&gt;
  &lt;name&gt;hbase.zookeeper.quorum&lt;/name&gt;
  &lt;value&gt;dev26i.vs.os.yandex.net,dev27i.vs.os.yandex.net,dev28i.vs.os.yandex.net&lt;/value&gt;
&lt;/property&gt;
&lt;/configuration&gt;</pre>

6. Create the /hbase directory in HDFS and set appropriate permissions

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -mkdir /hbase
$ sudo -u hdfs hadoop fs -chown hbase /hbase</pre>

7. Install region servers on node2 and node3

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo apt-get install hbase-regionserver</pre>

8. Start hbase-master on node1 and hbase-regionserver on node2 and node3

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ # NODE1
$ sudo service hbase-master start
$ # NODE2 & NODE3
$ sudo service hbase-regionserver start</pre>

9. Check HBase shell

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ hbase shell</pre>

At this point, I found that ZooKeeper was installed as three separate servers. When running the shell on a region server, I got these errors:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">2014-06-24 10:11:54,845 ERROR [main] client.HConnectionManager$HConnectionImplementation: The node /hbase is not in ZooKeeper. It should have been written by the master. Check the value configured in 'zookeeper.znode.parent'. There could be a mismatch with the one configured in the master.
2014-06-24 10:11:55,147 ERROR [main] client.HConnectionManager$HConnectionImplementation: The node /hbase is not in ZooKeeper. It should have been written by the master. Check the value configured in 'zookeeper.znode.parent'. There could be a mismatch with the one configured in the master.</pre>

Let's go back and check the ZooKeeper configuration:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ zookeeper-client -server dev26i.vs.os.yandex.net:2181
Connecting to dev26i.vs.os.yandex.net:2181
[zk: dev26i.vs.os.yandex.net:2181(CONNECTED) 1] ls /hbase
[meta-region-server, online-snapshot, replication, recovering-regions, splitWAL, rs, backup-masters, region-in-transition, draining, table, running, table-lock, master, hbaseid]

...
$ zookeeper-client -server dev27i.vs.os.yandex.net:2181
Connecting to dev27i.vs.os.yandex.net:2181
[zk: dev27i.vs.os.yandex.net:2181(CONNECTED) 0] ls /hbase
Node does not exist: /hbase
</pre>

I discovered that the configuration being used is /etc/zookeeper/conf/zoo.cfg. I updated it accordingly, then restarted ZooKeeper and HBase. Now everything appears to be working correctly.

To view the HBase master status, go to http://<hbase-master>:60010/master-status

 [1]: http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/CDH5-Installation-Guide/cdh5ig_hbase_install.html