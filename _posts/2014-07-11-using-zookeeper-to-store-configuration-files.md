---
title: Using ZooKeeper to store configuration files
author: Evgeny
layout: post
permalink: /2014-07-11-using-zookeeper-to-store-configuration-files/
categories:
  - Programming
tags:
  - Scala
  - Typesafe Config
  - ZooKeeper
---
ZooKeeper is not only the module required by others (like Kafka or HBase), but also useful tool for development. For example, you can easily store project configurations there. And it is quite easy to implement.

<!--more-->

We will use for that [Apache Curator][1] (ex NetFlix)

<pre class="toolbar:1 lang:default decode:true" title="Maven dependencies">...
        &lt;curator.version&gt;2.5.0&lt;/curator.version&gt;
        &lt;zookeeper.version&gt;3.4.5-cdh5.0.3&lt;/zookeeper.version&gt;
        &lt;typesafe-config.version&gt;1.2.1&lt;/typesafe-config.version&gt;
        &lt;commons-io.version&gt;1.3.2&lt;/commons-io.version&gt;
...
        &lt;dependency&gt;
            &lt;groupId&gt;org.apache.curator&lt;/groupId&gt;
            &lt;artifactId&gt;curator-framework&lt;/artifactId&gt;
            &lt;version&gt;${curator.version}&lt;/version&gt;
            &lt;exclusions&gt;
                &lt;exclusion&gt;
                    &lt;groupId&gt;org.apache.zookeeper&lt;/groupId&gt;
                    &lt;artifactId&gt;zookeeper&lt;/artifactId&gt;
                &lt;/exclusion&gt;
            &lt;/exclusions&gt;
        &lt;/dependency&gt;

        &lt;dependency&gt;
            &lt;groupId&gt;org.apache.zookeeper&lt;/groupId&gt;
            &lt;artifactId&gt;zookeeper&lt;/artifactId&gt;
            &lt;version&gt;${zookeeper.version}&lt;/version&gt;
            &lt;exclusions&gt;
                &lt;exclusion&gt;
                    &lt;groupId&gt;log4j&lt;/groupId&gt;
                    &lt;artifactId&gt;log4j&lt;/artifactId&gt;
                &lt;/exclusion&gt;
            &lt;/exclusions&gt;
        &lt;/dependency&gt;

        &lt;dependency&gt;
            &lt;groupId&gt;org.apache.commons&lt;/groupId&gt;
            &lt;artifactId&gt;commons-io&lt;/artifactId&gt;
            &lt;version&gt;${commons-io.version}&lt;/version&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;com.typesafe&lt;/groupId&gt;
            &lt;artifactId&gt;config&lt;/artifactId&gt;
            &lt;version&gt;${typesafe-config.version}&lt;/version&gt;
        &lt;/dependency&gt;
...</pre>

And simple client will be:

<pre class="toolbar:1 lang:scala decode:true" title="ConfigClient.scala">import java.io.{InputStream, File}

import org.apache.curator.framework.CuratorFrameworkFactory
import org.apache.curator.retry.ExponentialBackoffRetry

/**
 * Simple client to manage configuration files on specific base path
 * User: Evgeny Zhoga &lt;ezhoga@yandex-team.ru&gt;
 * Date: 11.07.14
 */
class ConfigClient(base: String) {
  private val curator = CuratorFrameworkFactory.newClient(
    ConfigFactory.load().getString("zookeeper.quorum"),
    new ExponentialBackoffRetry(10, 100))
  curator.start()

  /**
   * Put file to ZooKeeper
   * @param configNameInZookeeper name will be used to name config
   * @param configFileOnFS file on FS which will be put to ZooKeeper
   * @param overrideFlag if true, file in ZooKeeper will be overridden,
   *                     otherwise if exists in ZooKeeper then exception will be thrown. True is default
   */
  def put(
           configNameInZookeeper: String,
           configFileOnFS: File,
           overrideFlag: Boolean = true
           ) {
    put(configNameInZookeeper,
      configFileOnFS,
      overrideFlag,
      org.apache.commons.io.FileUtils.readFileToByteArray
    )
  }
  /**
   * Put file to ZooKeeper. same as
   * put(configNameInZookeeper: String,configFileOnFS: File,overrideFlag: Boolean)
   * with overrideFlag set to true
   * @param configNameInZookeeper name will be used to name config
   * @param streamToConfigFileOnFS file as InputStream which will be put to ZooKeeper
   */
  def put(
           configNameInZookeeper: String,
           streamToConfigFileOnFS: InputStream) {
    put(configNameInZookeeper, streamToConfigFileOnFS, overrideFlag = true)
  }
  /**
   * Put file to ZooKeeper
   * @param configNameInZookeeper name will be used to name config
   * @param streamToConfigFileOnFS file as InputStream which will be put to ZooKeeper
   * @param overrideFlag if true, file in ZooKeeper will be overridden,
   *                     otherwise if exists in ZooKeeper then exception will be thrown
   */
  def put(
           configNameInZookeeper: String,
           streamToConfigFileOnFS: InputStream,
           overrideFlag: Boolean
           ) {
    put(configNameInZookeeper,
      streamToConfigFileOnFS,
      overrideFlag,
      org.apache.commons.io.IOUtils.toByteArray: InputStream =&gt; Array[Byte]
    )
  }
  private def put[T] (
           configNameInZookeeper: String,
           configSource: T,
           overrideFlag: Boolean,
           toByteArray: T =&gt; Array[Byte]
           ) {
    val exsts = exists(configNameInZookeeper)
    if (!overrideFlag && exsts) {
      throw new RuntimeException(s"Override flag set to false and $configNameInZookeeper exists")
    }

    val data = toByteArray(configSource)
    if (exsts) {
      curator.setData().forPath(s"$base/$configNameInZookeeper", data)
    } else {
      curator.create().creatingParentsIfNeeded().forPath(s"$base/$configNameInZookeeper", data)
    }
  }

  /**
   * @param configNameInZookeeper name which will be checked
   * @return true if config name exists
   */
  def exists(configNameInZookeeper: String) = {
    curator.checkExists().forPath(s"$base/$configNameInZookeeper") != null
  }

  /**
   * Get config file from ZooKeeper and put it on FS
   * @param configNameInZookeeper config to get
   * @param configsLocation directory to store downloaded config
   * @param configFileNameOnFS new name for config on FS. Null by default, configNameInZookeeper used as name
   * @param overrideFlag true by default. If true and file on FS already exists it will be overridden.
   *                     If set to false and file exists, exception will be thrown
   */
  def get(
           configNameInZookeeper: String,
           configsLocation: File,
           configFileNameOnFS: String = null,
           overrideFlag: Boolean = true) = {
    val file = new File(configsLocation, Option(configFileNameOnFS).getOrElse(configNameInZookeeper))
    if (!overrideFlag && file.exists()) throw new RuntimeException(s"File ${file.getAbsolutePath} exists and not allowed for overriding")
    val data = curator.getData.forPath(s"$base/$configNameInZookeeper")
    org.apache.commons.io.FileUtils.writeByteArrayToFile(file, data)
    file
  }

  /**
   * Delete config from ZooKeeper
   * @param configName config to delete
   */
  def delete(configName: String) {
    curator.delete().forPath(s"$base/$configName")
  }

  def close() {
    curator.close()
  }
}</pre>

My application.conf

<pre class="toolbar:1 lang:default decode:true" title="application.conf">zookeeper {
  quorum = "dev26i.vs.os.yandex.net,dev27i.vs.os.yandex.net,dev28i.vs.os.yandex.net"
  bde {
    basenode = "/bde/properties"
  }
}
</pre>

Now, client usage is simple

<pre class="toolbar:2 nums:false lang:scala decode:true">// Put property file
val client = new ConfigClient(ConfigFactory.load().getString("zookeeper.bde.basenode"))
client.put("test.properties", new File("/path/to/property/file.txt"))
// Get property file
client.get("test.properties", new File("/path/to/properties/dir"), "property_file.name")
// And close it
client.close()</pre>

And so you have centralized storage for configurations.

 [1]: http://curator.apache.org/