---
title: 'Installing CDH5. ZooKeeper&#038;YARN'
author: Evgeny
layout: post
permalink: /2014-06-17-installing-cdh5-zookeeperyarn/
categories:
  - Big Data
tags:
  - CDH5
  - HDFS
  - Installation manual
  - YARN
  - ZooKeeper
---
I would try to install Cloudera CDH5 on cluster. Will use [CDH5 installation guide][1]

<!--more-->

**Preparations**

1. Install java7 on all nodes

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install yandex-jdk7
</pre>

alternatively [see][2]

2. add CDH5 repos

<pre class="toolbar:1 lang:sh decode:true" title="sudo vi /etc/apt/sources.list.d/cloudera.list">deb [arch=amd64] http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib
deb-src http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh precise-cdh5 contrib</pre>

3. Add a repo key

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ curl -s http://archive.cloudera.com/cdh5/ubuntu/precise/amd64/cdh/archive.key | sudo apt-key add -</pre>

4. Call update

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get update</pre>

(one of my nodes wrote huge amount of security errors, but as admins said &#8211; that&#8217;s ok, &#8220;this repo only for us&#8221;.. )

**Install and configure zookeeper**

1. Install

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install zookeeper
$ sudo apt-get install zookeeper-server</pre>

2. Create zookeeper configuration file. For example /var/lib/zookeeper/zookeeper.conf

<pre class="toolbar:1 lang:sh decode:true" title="sudo vi /var/lib/zookeeper/zookeeper.conf">tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=dev26i.vs.os.yandex.net:2888:3888
server.2=dev27i.vs.os.yandex.net:2888:3888
server.3=dev28i.vs.os.yandex.net:2888:3888

</pre>

3. Change owner:

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo chown -R zookeeper /var/lib/zookeeper/</pre>

4. Start each zookeeper (myid should be set accordingly step 6)

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo service zookeeper-server init --myid=1
$ sudo service zookeeper-server start
</pre>

5. Test zookeeper  
for example from node2 to node1:

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ zookeeper-client -server dev26i.vs.os.yandex.net:2181</pre>

**Install CDH5 packages and set roles.**

[YARN architecture][3]

*   Node1: Resource manager host, DataNode host, NodeManager host, MapReduce
*   Node2: NameNode host, DataNode host, NodeManager host, MapReduce
*   Node3: Secondary NameNode host, DataNode host, NodeManager host, MapReduce, History server, Proxy server

Node1

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-yarn-resourcemanager</pre>

Node2

<pre class="show-lang:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-hdfs-namenode</pre>

Node3

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-hdfs-secondarynamenode
...
Exception in thread "main" java.lang.IllegalArgumentException: Invalid URI for NameNode address (check fs.defaultFS): file:/// has no authority.
at org.apache.hadoop.hdfs.server.namenode.NameNode.getAddress(NameNode.java:350)
at org.apache.hadoop.hdfs.server.namenode.NameNode.getAddress(NameNode.java:338)
at org.apache.hadoop.hdfs.server.namenode.NameNode.getServiceAddress(NameNode.java:331)
at org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode.initialize(SecondaryNameNode.java:230)
at org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode.&lt;init&gt;(SecondaryNameNode.java:192)
at org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode.main(SecondaryNameNode.java:651)
invoke-rc.d: initscript hadoop-hdfs-secondarynamenode, action "start" failed.</pre>

ok. will see

On Data nodes (all servers in our cluster. But in big cluster resource manager can be separate)

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-yarn-nodemanager hadoop-hdfs-datanode hadoop-mapreduce</pre>

on one server (for example on third)

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-mapreduce-historyserver hadoop-yarn-proxyserver

Setting up hadoop-yarn-proxyserver (2.3.0+cdh5.0.1+567-1.cdh5.0.1.p0.46~precise-cdh5.0.1) ...
* Starting Hadoop proxyserver:
starting proxyserver, logging to /var/log/hadoop-yarn/yarn-yarn-proxyserver-dev28i.vs.os.yandex.net.out
invoke-rc.d: initscript hadoop-yarn-proxyserver, action "start" failed.

</pre>

on client hosts (for our case I will install on cluster&#8217;s servers)

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo apt-get install hadoop-client</pre>

Now deploying hosts:

**Deploying HDFS on a Cluster** (from [here][4])  
1. Copy the default configuration to your custom directory

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo cp -r /etc/hadoop/conf.empty /etc/hadoop/conf.my_cluster</pre>

2. Set alternatives:

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo update-alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.my_cluster 50
$ sudo update-alternatives --set hadoop-conf /etc/hadoop/conf.my_cluster</pre>

3. Update configuration files

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ cd /etc/hadoop/conf.my_cluster
</pre>

<pre class="lang:sh decode:true">&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;

&lt;configuration&gt;
&lt;property&gt;
&lt;name&gt;fs.defaultFS&lt;/name&gt;
&lt;value&gt;hdfs://dev27i.vs.os.yandex.net:8020&lt;/value&gt;
&lt;/property&gt;
&lt;/configuration&gt;
</pre>

<pre class="toolbar:1 lang:default decode:true" title="sudo vi hdfs-site.xml">&lt;configuration&gt;
  &lt;property&gt;
     &lt;name&gt;dfs.namenode.name.dir&lt;/name&gt;
     &lt;value&gt;file:///data/1/dfs/nn,file:///nfsmount/dfs/nn&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
     &lt;name&gt;dfs.datanode.data.dir&lt;/name&gt;
     &lt;value&gt;file:///data/1/dfs/dn,file:///data/2/dfs/dn,file:///data/3/dfs/dn,file:///data/4/dfs/dn&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
     &lt;name&gt;dfs.permissions.superusergroup&lt;/name&gt;
     &lt;value&gt;hadoop&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
    &lt;name&gt;dfs.datanode.failed.volumes.tolerated&lt;/name&gt;
    &lt;value&gt;2&lt;/value&gt;
  &lt;/property&gt;
&lt;/configuration&gt;</pre>

Create directories and set owner and permissions

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo mkdir -p /data/1/dfs/nn
$ sudo mkdir -p /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn
$ sudo chown -R hdfs:hdfs /data/1/dfs/nn /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn
$ sudo chmod 700 /data/1/dfs/nn /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn</pre>

See [Setup NFS][5] post to see how to setup NFS

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo -u hdfs mkdir -p /nfsmount/dfs/nn
$ sudo -u hdfs chmod 700 /nfsmount/dfs/nn</pre>

4. Format NameNode

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo -u hdfs hdfs namenode -format</pre>

5. Configure secondary NameNode

Add secondary namenode host (or hosts, one per line)

<pre class="toolbar:1 lang:sh decode:true" title="sudo vi /etc/hadoop/conf.my_cluster/masters">dev28i.vs.os.yandex.net</pre>

<pre class="toolbar:1 lang:default decode:true" title="sudo vi hdfs-site.xml">&lt;property&gt;
    &lt;name&gt;dfs.namenode.http-address&lt;/name&gt;
    &lt;value&gt;dev26i.vs.os.yandex.net:50070&lt;/value&gt;
    &lt;description&gt;
      The address and the base port on which the dfs NameNode Web UI will listen.
    &lt;/description&gt;
  &lt;/property&gt;</pre>

Set storage-balancing

<pre class="toolbar:1 lang:default decode:true" title="sudo vi hdfs-site.xml">&lt;property&gt;
    &lt;name&gt;dfs.datanode.fsdataset.volume.choosing.policy&lt;/name&gt;
    &lt;value&gt;org.apache.hadoop.hdfs.server.datanode.fsdataset.AvailableSpaceVolumeChoosingPolicy&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
    &lt;name&gt;dfs.datanode.available-space-volume-choosing-policy.balanced-space-threshold&lt;/name&gt;
    &lt;value&gt;10737418240&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
    &lt;name&gt;dfs.datanode.available-space-volume-choosing-policy.balanced-space-preference-fraction&lt;/name&gt;
    &lt;value&gt;0.75&lt;/value&gt;
  &lt;/property&gt;</pre>

**Start HDFS on cluster**

1. Configure folders on second and third nodes (data node folders)

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo mkdir -p /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn
$ sudo chown -R hdfs:hdfs /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn
$ sudo chmod 700 /data/1/dfs/dn /data/2/dfs/dn /data/3/dfs/dn /data/4/dfs/dn</pre>

2. Replicate configurations to other nodes:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ scp -r /etc/hadoop/conf.my_cluster dev27i.vs.os.yandex.net:/etc/hadoop/conf.my_cluster
$ scp -r /etc/hadoop/conf.my_cluster dev28i.vs.os.yandex.net:/etc/hadoop/conf.my_cluster</pre>

3. Setup alternatives

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo update-alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.my_cluster 50
$ sudo update-alternatives --set hadoop-conf /etc/hadoop/conf.my_cluster</pre>

4. Start hdfs

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ for x in `cd /etc/init.d ; ls hadoop-hdfs-*` ; do sudo service $x start ; done</pre>

5. Create /tmp dir on hdfs

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -mkdir /tmp
$ sudo -u hdfs hadoop fs -chmod -R 1777 /tmp</pre>

**Configure YARN**

1. Add to mapred-site.xml

<pre class="toolbar:1 lang:default decode:true" title="sudo vi mapred-site.xml">&lt;property&gt;
    &lt;name&gt;mapreduce.framework.name&lt;/name&gt;
    &lt;value&gt;yarn&lt;/value&gt;
  &lt;/property&gt;
</pre>

2. Setup yarn-site.xml

<pre class="toolbar:1 lang:default decode:true crayon-selected" title="sudo vi yarn-site.xml">&lt;configuration&gt;
  &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.hostname&lt;/name&gt;
    &lt;value&gt;dev26i.vs.os.yandex.net&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;name&gt;yarn.nodemanager.aux-services&lt;/name&gt;
    &lt;value&gt;mapreduce_shuffle&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;name&gt;yarn.nodemanager.aux-services.mapreduce_shuffle.class&lt;/name&gt;
    &lt;value&gt;org.apache.hadoop.mapred.ShuffleHandler&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;name&gt;yarn.log-aggregation-enable&lt;/name&gt;
    &lt;value&gt;true&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;description&gt;List of directories to store localized files in.&lt;/description&gt;
    &lt;name&gt;yarn.nodemanager.local-dirs&lt;/name&gt;
    &lt;value&gt;file:///data/1/yarn/local,file:///data/2/yarn/local,file:///data/3/yarn/local&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;description&gt;Where to store container logs.&lt;/description&gt;
    &lt;name&gt;yarn.nodemanager.log-dirs&lt;/name&gt;
    &lt;value&gt;file:///data/1/yarn/logs,file:///data/2/yarn/logs,file:///data/3/yarn/logs&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;description&gt;Where to aggregate logs to.&lt;/description&gt;
    &lt;name&gt;yarn.nodemanager.remote-app-log-dir&lt;/name&gt;
    &lt;!-- In Cloudera you can see: &lt;value&gt;hdfs://var/log/hadoop-yarn/apps&lt;/value&gt; but this is wrong --&gt;
    &lt;value&gt;/var/log/hadoop-yarn/apps&lt;/value&gt;
  &lt;/property&gt;

  &lt;property&gt;
    &lt;description&gt;Classpath for typical applications.&lt;/description&gt;
     &lt;name&gt;yarn.application.classpath&lt;/name&gt;
     &lt;value&gt;
        $HADOOP_CONF_DIR,
        $HADOOP_COMMON_HOME/*,$HADOOP_COMMON_HOME/lib/*,
        $HADOOP_HDFS_HOME/*,$HADOOP_HDFS_HOME/lib/*,
        $HADOOP_MAPRED_HOME/*,$HADOOP_MAPRED_HOME/lib/*,
        $HADOOP_YARN_HOME/*,$HADOOP_YARN_HOME/lib/*
     &lt;/value&gt;
  &lt;/property&gt;
&lt;/configuration&gt;</pre>

3. Setup folders and rights

<pre class="toolbar:2 nums:false lang:sh highlight:0 decode:true">$ sudo mkdir -p /data/1/yarn/local /data/2/yarn/local /data/3/yarn/local /data/4/yarn/local
$ sudo mkdir -p /data/1/yarn/logs /data/2/yarn/logs /data/3/yarn/logs /data/4/yarn/logs
$ sudo chown -R yarn:yarn /data/1/yarn/local /data/2/yarn/local /data/3/yarn/local /data/4/yarn/local
$ sudo chown -R yarn:yarn /data/1/yarn/logs /data/2/yarn/logs /data/3/yarn/logs /data/4/yarn/logs
</pre>

4. Configure history server in <samp class="ph codeph">mapred-site.xml</samp>

<pre class="toolbar:1 lang:default decode:true" title="sudo vi mapred-site.xml">&lt;property&gt;
    &lt;name&gt;mapreduce.jobhistory.address&lt;/name&gt;
    &lt;value&gt;dev28i.vs.os.yandex.net:10020&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
    &lt;name&gt;mapreduce.jobhistory.webapp.address&lt;/name&gt;
    &lt;value&gt;dev28i.vs.os.yandex.net:19888&lt;/value&gt;
  &lt;/property&gt;
</pre>

5. Setup proxing in <samp class="ph codeph">core-site.xml</samp>

<pre class="toolbar:1 lang:default decode:true" title="sudo vi core-site.xml">&lt;property&gt;
    &lt;name&gt;hadoop.proxyuser.mapred.groups&lt;/name&gt;
    &lt;value&gt;*&lt;/value&gt;
  &lt;/property&gt;
  &lt;property&gt;
    &lt;name&gt;hadoop.proxyuser.mapred.hosts&lt;/name&gt;
    &lt;value&gt;*&lt;/value&gt;
  &lt;/property&gt;</pre>

6. Add staging dir in mapred-site.xml

<pre class="toolbar:1 lang:default decode:true" title="sudo vi mapred-site.xml">&lt;property&gt;
    &lt;name&gt;yarn.app.mapreduce.am.staging-dir&lt;/name&gt;
    &lt;value&gt;/user&lt;/value&gt;
&lt;/property&gt;</pre>

7. Sync configurations over cluster

8. Create history directory

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -mkdir -p /user/history
$ sudo -u hdfs hadoop fs -chmod -R 1777 /user/history
$ sudo -u hdfs hadoop fs -chown mapred:hadoop /user/history
</pre>

9. Create log directory

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -mkdir -p /var/log/hadoop-yarn
$ sudo -u hdfs hadoop fs -chown yarn:mapred /var/log/hadoop-yarn</pre>

10. Verifying file structure:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -ls -R /

drwxrwxrwt   - hdfs hadoop          0 2014-06-17 08:52 /tmp
drwxr-xr-x   - hdfs hadoop          0 2014-06-17 11:56 /user
drwxrwxrwt   - mapred hadoop          0 2014-06-17 11:56 /user/history
drwxr-xr-x   - hdfs   hadoop          0 2014-06-17 11:58 /var
drwxr-xr-x   - hdfs   hadoop          0 2014-06-17 11:58 /var/log
drwxr-xr-x   - yarn   mapred          0 2014-06-17 11:58 /var/log/hadoop-yarn</pre>

11. Start/restart resource manager

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo service hadoop-yarn-nodemanager start
or
$ sudo service hadoop-yarn-nodemanager restart</pre>

12. Start/restart all NodeManagers

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo service hadoop-yarn-nodemanager start
or
$ sudo service hadoop-yarn-nodemanager restart</pre>

13. Start/restart history server

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo service hadoop-mapreduce-historyserver start
or
$ sudo service hadoop-mapreduce-historyserver restart</pre>

14. Add current user:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true">$ sudo -u hdfs hadoop fs -mkdir /user/$USER
$ sudo -u hdfs hadoop fs -chown $USER /user/$USER</pre>

&nbsp;

Looks working

 [1]: http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/CDH5-Installation-Guide/CDH5-Installation-Guide.html
 [2]: https://help.ubuntu.com/community/Java#Oracle_Java_7
 [3]: http://hadoop.apache.org/docs/r2.3.0/hadoop-yarn/hadoop-yarn-site/YARN.html
 [4]: http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/CDH5-Installation-Guide/cdh5ig_hdfs_cluster_deploy.html#topic_11_2_2_unique_1
 [5]: http://blog.forteamwork.ru/?p=39