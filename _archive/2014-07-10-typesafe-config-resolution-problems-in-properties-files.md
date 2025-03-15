---
title: Typesafe Config resolution problems in properties files
author: Evgeny
layout: post
permalink: /2014-07-10-typesafe-config-resolution-problems-in-properties-files/
categories:
  - Programming
tags:
  - Scala
  - Typesafe Config
---
We have many configuration files in our projects. Some of them are common - shared across all projects in the department, while others are specific to server groups. Common configurations are stored in properties files. Some projects use Typesafe [Config][1], while others use only [Spring][2] properties loader.

<!--more-->

In our project, we use Typesafe Config. Previously, our properties were just plain values (resulting in many duplicate server names in common properties). Recently, our admins decided to change this approach and started using placeholders. However, we discovered that when using Typesafe Config with two layers (with properties files containing placeholders at the bottom), the placeholders weren't being resolved.

After several attempts (reading Config sources and searching online), I decided to write a simple resolver since I couldn't find any working solution.

<pre class="toolbar:2 lang:scala decode:true">object ConfigResolver {
  val regex = "(\\$\\{([\\w][\\w\\d_\\.\\-]*)\\}|\\$([\\w][\\w\\d_]*))".r

  def apply(original: Config, absentPlaceholders:scala.collection.mutable.HashSet[String] = new scala.collection.mutable.HashSet[String]): Config = {
    // we start to fill empty Config with resolved placeholders
    val resolved = original.entrySet().foldLeft(ConfigFactory.empty())((r, entry) => {
      // only in case if type of value is string
      if (entry.getValue.valueType() == ConfigValueType.STRING) {
        val value = entry.getValue.unwrapped().toString
        // we look for ${some.val.with-dash.or_underscore.or.numbers123}
        // or $some_val_with_underscore_and_numbers123
        // Is it right regexp??
        val newval = regex.
          findAllMatchIn(value).
          //then replace all occurences with values from config
          foldLeft(value)(
            (v, m) => {
              val full = m.group(1)
              val next = Option(m.group(2)).getOrElse(m.group(3))
              //but only if we previously did not detected that we do not have this placeholder in Config
              if (!absentPlaceholders.contains(next)) {
                // and only in case we find it in Config
                if (original.hasPath(next)) {
                  v.replaceAllLiterally(full, original.getString(next))
                } else {
                  // and if not - we cache it in absent paths
                  absentPlaceholders.add(next)
                  v
                }
              } else v
            }
          )

        //if newval is different from value we return Config with new value for key
        if (value != newval) r.withValue(entry.getKey, ConfigValueFactory.fromAnyRef(newval))
        //otherwise we return same Config 
        else r
      } else {
        // also same config if type is not string
        r
      }
    })
    // if new Config with resolved values is empty - we return same Config
    if (resolved.entrySet().isEmpty) {
      // as soon as we finished - we clear this set
      absentPlaceholders.clear()
      original
    }
    // otherwise we try to make another iteration with "resolution config" with fallback by original
    else apply(resolved.withFallback(original), absentPlaceholders)

  }
}</pre>

Now the loading happens like this:

<pre class="toolbar:2 lang:scala decode:true">val p = {
    val conf = ConfigFactory.load()
    ConfigResolver(
      getLocaleName.fold(conf)(
        specific =>
          ConfigFactory.load(
            ConfigFactory.parseFile(new File(s"/etc/server/$specific/file.properties")).
              withFallback(conf)
          )))}</pre>

&nbsp;

If you know something about this problem that I'm not aware of, please let me know! =)

 [1]: https://github.com/typesafehub/config
 [2]: http://spring.io