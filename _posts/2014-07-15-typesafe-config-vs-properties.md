---
title: Typesafe config VS properties or keys VS paths
author: Evgeny
layout: post
permalink: /2014-07-15-typesafe-config-vs-properties/
categories:
  - Programming
tags:
  - Typesafe Config
---
I recently encountered problems with properties files loaded into Typesafe Config.

In properties files, this syntax is acceptable:

<pre class="toolbar:1 lang:default decode:true" title="some.properties">foo.bar.var1=123
foo.bar.var2=321
foo.bar=${foo.bar.var2}</pre>

This works because properties files are simply a mapping of {key => value}.

However, for Typesafe Config, it's different. When loading, first an object **foo.bar** will be created with two fields: [var1, var2]. And then this object will be overridden with the string "${foo.bar.var2}".