---
layout: post
title: About muCommander
---

Ten years ago (2008) I submitted my first contribution to an open source project named [muCommander](http://ahadas.github.io/about/#mucommander) that I maintain to this day. This post provides a short description of the project, the recent changes we have made, challenges we are facing, and some future plans.

# What is muCommander
_<b>muCommander is a lightweight, cross-platform file manager with a dual-pane interface. It runs on any operating system with Java support (Mac OS X, Windows, Linux, *BSD, Solaris...).</b>_  

In other words, muCommander is a long-standing (since 2002) open-source (GPLv3) file manager with a dual-pane interface (similar to that of [Norton Commander](https://en.wikipedia.org/wiki/Norton_Commander)) that can run on all the mainstream operating systems.

# Why muCommander
First and foremost, muCommander supports various file formats (e.g., ZIP) and protocols (e.g., SMB). It complements common built-in file managers with additional file formats, like 7z, and capabilities, like on-the-fly editing of ZIP files in Mac OS X. Moreover, it eliminates the need to set up protocol-specific clients, like FTP-clients.

And not only that muCommander supports many file protocols but it also abstracts reads and writes with Java input/output streams. That way, reading from one remote file protocol and writing to a second remote file protocol can be made very efficient. For instance, it can read files from an FTP server and write them to an SMB server without writing them to a third location. It is done by reading a portion of the file from an input-stream connected to the FTP server and immediately write it to an output-stream connected to the SMB server.  

These great capabilities are provided on all mainstream operating systems (OS). By having its core and most of its functionality written in Java, muCommander is made cross-platform. Except for some OS-specific features that use native code (e.g., moving files to the trash), everything is implemented in the Java language. For developers it can be significant as it conforms the principle of _write once, run anywhere_.  

Another benefit of being implemented in Java is that muCommander can leverage the large variety of third-party client-side code for different file protocols.  

# Recent changes
Honestly, the [pulse of the project](https://github.com/mucommander/mucommander/pulse) was not that great recently. I will touch that later on. Nevertheless, we have done some important changes.

## Technical
* Converted the code repository to Git.
* Moved the project to [GitHub](https://github.com/mucommander).
* Replaced the build system with Gradle.
* [Enable compiling the code with Java 9 and Java 10](https://github.com/mucommander/mucommander/pull/158). 
* Fix various bugs.

## Communal
* Updated the [website](http://www.mucommander.com).
* Revived the [twitter account](https://twitter.com/mucommander).
* Use [GITTER](https://gitter.im/mucommander/Lobby).

# Challenges
* libguestfs
* vsphere
* scale the development
* getting feedback from the community
* competitive - non open source, mac os x specific

# Future plans