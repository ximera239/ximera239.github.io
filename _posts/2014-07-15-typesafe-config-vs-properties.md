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
Suddenly got problems with properties files loaded into typesafe config.

In properties such syntax is ok:

<pre class="toolbar:1 lang:default decode:true" title="some.properties">foo.bar.var1=123
foo.bar.var2=321
foo.bar=${foo.bar.var2}</pre>

Just because it is a mapping {key => value}

But for typesafe config it is different. When loading &#8211; at first object **foo.bar **will be created with two fields: [var1, var2]. And then object will be overridden with string &#8220;${foo.bar.var2}&#8221;.