---
title: Installing Octave on OSX 10.9.5
author: Evgeny
layout: post
permalink: /2014-10-21-installing-octave-on-osx-10-9-5/
categories:
  - Learning
tags:
  - coursera
  - ML
---
During coursera course about Machine Learning I tried to install Octave (3.8.1) with Homebrew.

<!--more-->

But I had some troubles. I used [this][1] instruction

1. I used to run

<pre class="toolbar:2 nums:false lang:default decode:true">$ brew unlink libtool && brew link libtool</pre>

Probably because brew installations have broken links after OSX upgrade from 10.8

2. I need to install openblas (I got errors about BLAS on octave installation: &#8220;configure: error: A BLAS library was detected but found incompatible with your Fortran 77 compiler settings.&#8221;)

<pre class="toolbar:2 nums:false lang:default decode:true ">$ brew install openblas</pre>

3. To run installation of octave with another blas lib (which does not override default mac installation) I added another parameter:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true ">$ brew install octave --without-docs --with-openblas</pre>

And I got it at the end. Probably this helps somebody..

 [1]: http://wiki.octave.org/Octave_for_MacOS_X#Binary_installer_for_OSX_10.9.1