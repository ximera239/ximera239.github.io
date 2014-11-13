---
title: Kafka logback appender
author: Evgeny
layout: post
permalink: /2014-08-19-kafka-logback-appender/
categories:
  - Programming
tags:
  - Kafka
  - Scala
---
This part is very simple.

<!--more-->

> Really simple? Or just now, after vacation? If I remember well &#8211; I spent some time to make it work..

We need serializers for key&value, partitioner, key class and appender itself. For my example I used LoggingEventVO class to serialize value (I would say that protobuf model is better alternative for serialization). Key &#8211; just our host who send message.

<pre class="toolbar:1 lang:scala decode:true" title="ValueEncoder.scala">package ru.forteamwork.examples.kafka.logback.logger

import java.io.{ByteArrayOutputStream, ObjectOutputStream}

import ch.qos.logback.classic.spi.{ILoggingEvent, LoggingEventVO}
import kafka.utils.VerifiableProperties

class ValueEncoder(props: VerifiableProperties = null) extends scala.AnyRef with kafka.serializer.Encoder[ILoggingEvent] {
  override def toBytes(t: ILoggingEvent): Array[Byte] = {
    val output = new ByteArrayOutputStream()
    val objectOutput = new ObjectOutputStream(output)
    objectOutput.writeObject(LoggingEventVO.build(t))
    val result = output.toByteArray
    try {
      output.close()
      objectOutput.close()
    }
    result
  }
}
</pre>

Key class:

<pre class="toolbar:1 lang:scala decode:true" title="Key.scala">package ru.forteamwork.examples.kafka.logback

case class Key(host: String)
</pre>

And Key serializer

<pre class="toolbar:1 lang:scala decode:true " title="KeyEncoder.scala">package ru.forteamwork.examples.kafka.logback.logger

import kafka.utils.VerifiableProperties
import ru.forteamwork.examples.kafka.logback

class KeyEncoder(props: VerifiableProperties = null) extends scala.AnyRef with kafka.serializer.Encoder[logback.Key] {
  override def toBytes(t: logback.Key): Array[Byte] = {
    t.host.getBytes
  }
}
</pre>

Partitioner

<pre class="toolbar:1 lang:scala decode:true " title="Partitioner.scala">package ru.forteamwork.examples.kafka.logback.logger

import kafka.producer.{Partitioner =&gt; P}
import kafka.utils.VerifiableProperties
import ru.forteamwork.examples.kafka.logback

class Partitioner(props: VerifiableProperties = null) extends P() {

  def partition(key: Any, numPartitions: Int) = {
    math.abs(key.asInstanceOf[logback.Key].host.hashCode) % numPartitions
  }
}
</pre>

And, at last, appender

<pre class="toolbar:1 lang:scala decode:true " title="KafkaAppender.scala">package ru.forteamwork.examples.kafka.logback

import java.util.Properties

import ch.qos.logback.classic.spi.ILoggingEvent
import ch.qos.logback.core.AppenderBase
import kafka.javaapi.producer.Producer
import kafka.producer.{KeyedMessage, ProducerConfig}

class KafkaAppender extends AppenderBase[ILoggingEvent] {

  @reflect.BeanProperty var topic: String = _
  @reflect.BeanProperty var brokers: String = _
  private var producer: Producer[Key, ILoggingEvent] = _
  private val getHostname = Option(System.getProperty("host.name")) getOrElse(throw new RuntimeException("host.name should be set"))

  override def start() {
    super.start()
    val props = new Properties
    props.put("metadata.broker.list", brokers)
    props.put("serializer.class", classOf[logger.ValueEncoder].getName)
    props.put("key.serializer.class", classOf[logger.KeyEncoder].getName)
    props.put("partitioner.class", classOf[logger.Partitioner].getName)
    props.put("request.required.acks", "1")

    val config = new ProducerConfig(props)
    producer = new Producer[Key, ILoggingEvent](config)
  }
  override def stop() {
    super.stop()
    producer.close
  }

  protected def append(event: ILoggingEvent) {
    producer.send(
      new KeyedMessage[Key, ILoggingEvent](
        this.topic,
        Key(getHostname),
        event)
    )
  }

}
</pre>

Using of appender will be:

<pre class="toolbar:1 nums:false lang:default decode:true" title="part of logback.xml">...
    &lt;appender name="LOG_KAFKA" class="ru.forteamwork.examples.kafka.logback.KafkaAppender"&gt;
        &lt;topic&gt;example-logs-topic&lt;/topic&gt;
        &lt;brokers&gt;your kafka brokers should be here..&lt;/brokers&gt;
    &lt;/appender&gt;

...</pre>

&nbsp;