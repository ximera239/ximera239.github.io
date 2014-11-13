---
title: Another Kafka syslog server
author: Evgeny
layout: post
permalink: /2014-08-19-another-kafka-syslog-server/
categories:
  - Programming
tags:
  - Kafka
  - Scala
---
Some time ago I saw on [this][1] (now it is dead) page &#8220;possible project&#8221; about storing syslog messages in Kafka. Quite fast I found [this][2] implementation.

<!--more-->

First of all I rewrote it to scala. During this process I found that project was written for early kafka release, so some fixes were required.

So protobuf model is the same. Other parts:

Encoder and decoder:

<pre class="toolbar:1 lang:default decode:true " title="SyslogMessageEncoder.scala">package ru.forteamwork.examples.kafka.syslog.serialize

import kafka.serializer.Encoder
import ru.forteamwork.examples.kafka.syslog.SyslogProto.SyslogMessage

class SyslogMessageEncoder(props : kafka.utils.VerifiableProperties = null) extends Encoder[SyslogMessage] {
  def toBytes(smsg: SyslogMessage) = {
    smsg.toByteArray
  }
}
</pre>

&nbsp;

<pre class="toolbar:1 lang:scala decode:true ">package ru.forteamwork.examples.kafka.syslog.serialize

import com.google.protobuf.InvalidProtocolBufferException
import kafka.serializer.Decoder
import org.slf4j.LoggerFactory
import ru.forteamwork.examples.kafka.syslog.SyslogProto.SyslogMessage
import ru.forteamwork.examples.kafka.syslog.SyslogProto

class SyslogMessageDecoder(props : kafka.utils.VerifiableProperties = null) extends Decoder[SyslogMessage] {
  private val log = LoggerFactory.getLogger(getClass)

  def fromBytes(bytes : Array[Byte]) = {
    try {
      SyslogProto.SyslogMessage.parseFrom(bytes)
    } catch {
      case e: InvalidProtocolBufferException =&gt;
        log.error("Received unparseable message", e)
        null
    }
  }
}
</pre>

Event handler

<pre class="toolbar:1 lang:scala decode:true " title="KafkaEventHandler.scala">package ru.forteamwork.examples.kafka.syslog

import java.net.SocketAddress

import kafka.javaapi.producer.Producer
import kafka.producer.KeyedMessage
import org.productivity.java.syslog4j.server.{SyslogServerEventIF, SyslogServerIF, SyslogServerSessionlessEventHandlerIF}
import org.slf4j.LoggerFactory
import ru.forteamwork.examples.kafka.syslog.SyslogProto.SyslogMessage
import ru.forteamwork.examples.kafka.syslog.SyslogProto.SyslogMessage.Severity


class KafkaEventHandler(producer: Producer[String, SyslogMessage]) extends SyslogServerSessionlessEventHandlerIF {
  private val serialVersionUID = 1797629243068715681L
  private val log = LoggerFactory.getLogger(getClass)

  def event(server: SyslogServerIF, socketAddress: SocketAddress, event: SyslogServerEventIF) {
    val smb = SyslogMessage.newBuilder()
    // if there wasn't a timestamp in the event for any reason use now
    val ts = Option(event.getDate).map(_.getTime).getOrElse(System.currentTimeMillis)
    smb.setTimestamp(ts)
    smb.setFacility(event.getFacility)
    smb.setSeverity(Severity.valueOf(event.getLevel))
    smb.setHost(event.getHost)
    smb.setMsg(event.getMessage.trim)
    send(smb.build())
  }

  def exception(server: SyslogServerIF, socketAddress: SocketAddress, exception: Exception) {
    log.warn("exception", exception)
    // ignore
  }

  def initialize(syslogServer: SyslogServerIF) {}

  def destroy(syslogServer: SyslogServerIF) {}

  private def send(message: SyslogMessage) {
    val pd = new KeyedMessage[String, SyslogMessage]("syslog-kafka", message)
    producer.send(pd)
  }
}
</pre>

And server.

<pre class="toolbar:1 lang:scala decode:true" title="SyslogKafkaServer.scala">package ru.forteamwork.examples.kafka.syslog

import java.util.Properties

import kafka.javaapi.producer.Producer
import kafka.producer.ProducerConfig
import org.productivity.java.syslog4j.server.impl.net.udp.UDPNetSyslogServerConfig
import org.slf4j.LoggerFactory
import ru.forteamwork.examples.Environment
import ru.forteamwork.examples.kafka.syslog.SyslogProto.SyslogMessage
import ru.forteamwork.examples.kafka.syslog.UDPServer

import scala.collection.JavaConversions._


object SyslogKafkaServer {
  private val log = LoggerFactory.getLogger(getClass)

  def getDefaultKafkaProperties: Properties = {
    val props = new Properties()
    for (entry &lt;- Environment.config.getConfig("syslog.kafka").entrySet()) {
      props.put(entry.getKey, entry.getValue.unwrapped().toString)
    }
    props
  }

  def main(args: Array[String]) {
    val kafkaProperties = getDefaultKafkaProperties
    val config = new ProducerConfig(kafkaProperties)
    val producer = new Producer[String, SyslogMessage](config)
    val kafkaEventHandler = new KafkaEventHandler(producer)

    val syslogConfig = new UDPNetSyslogServerConfig(
      Environment.config.getString("syslog.server.host"),
      Environment.config.getInt("syslog.server.port"))
    syslogConfig.addEventHandler(kafkaEventHandler)

    val syslogServer = new UDPServer(syslogConfig)

    Runtime.getRuntime.addShutdownHook(new Thread() {
      override def run() {
        if (producer != null) {
          log.info("Closing producer...")
          producer.close
        }
      }
    })
  }
}
</pre>

In server I tried to add some akka flavor &#8211; I added simple udp server implementation based on akka instead of using org.productivity.java.syslog4j.server.SyslogServer. It is simple. Has just two actors &#8211; one catchs UGP message and send it to another actor, who process data to kafka.

<pre class="toolbar:1 lang:scala decode:true" title="Worker.scala">package ru.forteamwork.examples.kafka.syslog.udp

import java.net.InetSocketAddress

import akka.actor.{Props, Actor, ActorRef}
import akka.io.{Udp, IO}
import org.productivity.java.syslog4j.SyslogConstants
import org.productivity.java.syslog4j.server.impl.net.udp.UDPNetSyslogServerConfig
import ru.forteamwork.examples.kafka.syslog.DataWorker
import ru.forteamwork.examples.kafka.syslog.message.SLMessage

class Worker(config: UDPNetSyslogServerConfig) extends Actor {

  def this() = this(new UDPNetSyslogServerConfig(SyslogConstants.SYSLOG_HOST_DEFAULT))
  import context.system

  val dataWorker = context.actorOf(Props(classOf[DataWorker], config), "syslog-worker")

  IO(Udp) ! Udp.Bind(self, new InetSocketAddress(config.getHost, config.getPort))

  def receive = {
    case Udp.Bound(local) =&gt;
      context.become(ready(sender()))
  }

  def ready(socket: ActorRef): Receive = {
    case Udp.Received(data, remote) =&gt;
      dataWorker ! SLMessage(data, remote)
    case Udp.Unbind =&gt; socket ! Udp.Unbind
    case Udp.Unbound =&gt; context.stop(self)
  }
}

</pre>

&nbsp;

<pre class="toolbar:1 lang:scala decode:true" title="DataWorker.scala">package ru.forteamwork.examples.kafka.syslog

import java.net.InetAddress

import akka.actor.Actor
import org.productivity.java.syslog4j.SyslogCharSetIF
import org.productivity.java.syslog4j.server.impl.event.SyslogServerEvent
import org.productivity.java.syslog4j.server.impl.event.structured.StructuredSyslogServerEvent
import org.productivity.java.syslog4j.server.impl.net.udp.UDPNetSyslogServerConfig
import org.productivity.java.syslog4j.server.{SyslogServerSessionEventHandlerIF, SyslogServerSessionlessEventHandlerIF, SyslogServerEventIF, SyslogServerConfigIF}
import org.productivity.java.syslog4j.util.SyslogUtility
import ru.forteamwork.examples.kafka.syslog.message.SLMessage

import scala.collection.JavaConversions._

class DataWorker(config: UDPNetSyslogServerConfig) extends Actor {
  def receive = {
    case SLMessage(data, remote) =&gt;
      val dataForJava = data.toArray[Byte]
      val processed = DataWorker.createEvent(config, dataForJava, dataForJava.length, remote.getAddress)
      Option(processed.getHost).foreach(host =&gt; {
        for (rawhandler &lt;- config.getEventHandlers) {
          rawhandler match {
            case handler: SyslogServerSessionlessEventHandlerIF =&gt;
              handler.event(null, remote, processed)
            case handler: SyslogServerSessionEventHandlerIF =&gt;
              handler.event(null, null, remote, processed)
          }
        }
      })

  }
}

object DataWorker {
  def createEvent(serverConfig: SyslogServerConfigIF,
                  lineBytes: Array[Byte],
                  lineBytesLength: Int,
                  inetAddr: InetAddress) = {
    var event:SyslogServerEventIF = null

    if (serverConfig.isUseStructuredData && isStructuredMessage(serverConfig,lineBytes)) {
      val event = new StructuredSyslogServerEvent(lineBytes,lineBytesLength,inetAddr)
      if (serverConfig.getDateTimeFormatter != null) {
        event.asInstanceOf[StructuredSyslogServerEvent].setDateTimeFormatter(serverConfig.getDateTimeFormatter)
      }
      event
    } else {
      new SyslogServerEvent(lineBytes,lineBytesLength,inetAddr)
    }
  }
  def isStructuredMessage(syslogCharSet: SyslogCharSetIF, receiveData: Array[Byte]) =
    isStructuredMessage0(syslogCharSet,SyslogUtility.newString(syslogCharSet, receiveData))

  def isStructuredMessage0(syslogCharSet: SyslogCharSetIF, receiveData: String) = {
    val idx = receiveData.indexOf('&gt;')
    // If there's a numerical VERSION field after the &lt;priority&gt;, return true.
    idx != -1 && receiveData.length() &gt; idx + 1 && Character.isDigit(receiveData.charAt(idx + 1))
  }
}</pre>

And server itself

<pre class="toolbar:1 lang:scala decode:true">package ru.forteamwork.examples.kafka.syslog

import akka.actor.{Props, ActorSystem}
import org.productivity.java.syslog4j.server.impl.net.udp.UDPNetSyslogServerConfig
import ru.forteamwork.examples.kafka.syslog.udp.Worker

class UDPServer(config: UDPNetSyslogServerConfig) {
  val system = ActorSystem("syslog-udp-server")
  val worker = system.actorOf(Props(classOf[Worker], config), "udp-worker")
}
</pre>

&nbsp;

 [1]: http://kafka.apache.org/projects.html
 [2]: https://github.com/xstevens/syslog-kafka