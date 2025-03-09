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
During a Coursera course on Machine Learning, I tried to install Octave (3.8.1) with Homebrew.

<!--more-->

However, I encountered some issues. I followed [this][1] instruction with some modifications:

1. I had to run:

<pre class="toolbar:2 nums:false lang:default decode:true">$ brew unlink libtool && brew link libtool</pre>

This was likely necessary because brew installations have broken links after upgrading OSX from 10.8.

2. I needed to install openblas (I received errors about BLAS during Octave installation: "configure: error: A BLAS library was detected but found incompatible with your Fortran 77 compiler settings.")

<pre class="toolbar:2 nums:false lang:default decode:true ">$ brew install openblas</pre>

3. To run the installation of Octave with another BLAS library (which doesn't override the default Mac installation), I added another parameter:

<pre class="toolbar:2 nums:false lang:default highlight:0 decode:true ">$ brew install octave --without-docs --with-openblas</pre>

This eventually worked for me. Hopefully this helps someone else.

 [1]: http://wiki.octave.org/Octave_for_MacOS_X#Binary_installer_for_OSX_10.9.1