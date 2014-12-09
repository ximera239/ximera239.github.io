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
During Octave tutorial I got an error:

        octave:> hist(w)

        gnuplot> set terminal aqua enhanced title "Figure 1" size 560 420  font "*,6" dashlength 1
                      ^
                 line 0: unknown or ambiguous terminal type; type just 'set terminal' for a list

[This][1] helps me. See answer from mackuntu.

~~~ bash
$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/AquaTerm /usr/local/lib/libaquaterm.dylib
$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/AquaTerm /usr/local/lib/libaquaterm.1.0.0.dylib
$ sudo mkdir /usr/local/include/aquaterm
$ sudo ln -s /Library/Frameworks/AquaTerm.framework/Versions/A/Headers/* /usr/local/include/aquaterm/.
$ brew install gnuplot --with-aquaterm
~~~

&nbsp;

 [1]: http://stackoverflow.com/questions/13786754/octave-gnuplot-aquaterm-error-set-terminal-aqua-enhanced-title-figure-1-unk