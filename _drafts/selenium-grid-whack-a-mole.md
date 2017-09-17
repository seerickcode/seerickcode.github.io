---
layout: post
title: Selenium Grid - Whack-a-mole
modified:
categories: devops monitoring 
description:
tags: [selenium]
image:
  feature:
  credit:
  creditlink:
comments: True
share: True
---

Write about a Selenium Grid setup

What is Selinum Grid ?

Server
-Going to use Saltstack to deploy - Try and find a forumla , probably garbage, but a good base.
-Nothing
-Found seleniumHQ has docker , but we are not going to be using docker, or LXC.  I am making
a 'pet' server (pet vs. cattle).  It is going to be long running, permanent, and if something
goes wrong, I need to figure out why, not just shoot it in the head.  Also, don't want ubuntu base, 
don't want a base that has client parts in it (browsers, etc.).

Formula - Insert about how to properly write a formula, but start with a basic.
Going to use the code in the SeleniumHQ Docker/Hub Dockerfile to see what needs to be done.

Hub derives from SeleniumHQ Docker/Base, so off to that Docker file.

See some basic tools (sudo, unzip, wget, etc.) already will be handled by my Saltstack deploy.
Java - Headless is in there - We will need that, but I also have a Java Saltstack formula, so
I will leverage that.

Looks like Selenium itself (for docker) is just dumped from a wget into /opt/selenium and some user setup.
Let's dig in.

Spin up a minimal Debian 8 container, get salt minion on there, and do a default run.




