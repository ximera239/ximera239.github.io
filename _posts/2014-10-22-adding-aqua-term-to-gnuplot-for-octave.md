---
title: Adding aqua term to gnuplot (for octave)
author: Evgeny
layout: post
permalink: /2014-10-22-adding-aqua-term-to-gnuplot-for-octave/
categories:
  - Learning
tags:
  - coursera
  - ML
---
During Octave tutorial got an error:

<!--more-->

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true ">octave:2&gt; hist(w)

gnuplot&gt; set terminal aqua enhanced title "Figure 1" size 560 420  font "*,6" dashlength 1
                      ^
         line 0: unknown or ambiguous terminal type; type just 'set terminal' for a list
</pre>

[This][1] helps me. Answer from mackuntu

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true ">$ brew uninstall gnuplot

$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/AquaTerm /usr/local/lib/libaquaterm.dylib
$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/AquaTerm /usr/local/lib/libaquaterm.1.0.0.dylib
$ sudo mkdir /usr/local/include/aquaterm
$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/Headers/* /usr/local/include/aquaterm/.
$ brew install gnuplot --with-aquaterm</pre>

&nbsp;

 [1]: http://stackoverflow.com/questions/13786754/octave-gnuplot-aquaterm-error-set-terminal-aqua-enhanced-title-figure-1-unk