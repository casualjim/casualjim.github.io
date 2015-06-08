---
date: 2012-04-24T23:46:04Z
title: searchable sbt wiki
comments: true
categories:
- Scala
- sbt
tags:
- scala
- sbt
---
## Searchable sbt wiki published

Last week I went to [Scala Days](http://days2012.scala-lang.org/) and had a blast. I got to meet many of the people whose software I use every day.  
Most Scala developers I know of use sbt to build scala projects.
SBT has a great [wiki](https://github.com/harrah/xsbt/wiki) with loads of information but is hosted on github and they, unfortunately, have no search box on their wikis that will search just the wiki.  This leads to long searches on the sbt wiki or grepping the checked out git repo or worse.
However the [gollum](https://github.com/github/gollum) software which runs the github wikis does have a search box, so I decided to host the sbt wiki as a read-only version somewhere else.

The same server that hosts our jenkins frontend is now also hosting the [sbt wiki](https://sbtwiki.backchat.io) and the main feature is *a search box*. It's running a unicorn server for rack fronted by nginx for ssl termination.

Enjoy the new [sbt wiki](https://sbtwiki.backchat.io).
