---
layout: post
title: About muCommander
---

Ten years ago (2008) I submitted my first contribution to an open source project named [muCommander](http://ahadas.github.io/about/#mucommander) that I maintain to this day. This post provides a short description of the project, the recent changes we have made, challenges we are facing, and some future plans.

# What?
_**muCommander is a lightweight, cross-platform file manager with a dual-pane interface. It runs on any operating system with Java support (Mac OS X, Windows, Linux, *BSD, Solaris...).**_  

In other words, muCommander is a long-standing (since 2002) open-source (GPLv3) file manager with a dual-pane interface (similar to that of [Norton Commander](https://en.wikipedia.org/wiki/Norton_Commander)) that can run on all the mainstream operating systems.

# Why?
First and foremost, muCommander supports various file formats (e.g., ZIP) and protocols (e.g., SMB). It complements common built-in file managers with additional file formats, like 7z, and capabilities, like on-the-fly editing of ZIP files on Mac OS X. Moreover, it eliminates the need to set up protocol-specific clients, such as an FTP-client.

And not only that muCommander supports many file protocols but it also abstracts reads and writes with Java input/output streams. That way, reading from a remote file protocol and writing to another remote file protocol can be made very efficient. For instance, it can read files from an FTP server and write them to an SMB server without writing them to a temporary persistent location. That is done by reading a portion of the file from an input-stream connected to the FTP server and immediately write it to an output-stream connected to the SMB server.  

These great capabilities are provided on all mainstream operating systems (OS). By having its core and most of its functionality written in Java, muCommander becomes cross-platform. Except for some OS-specific features that use native code (e.g., moving files to the trash), everything is implemented in the Java language. For developers it is significant as it conforms the principle of _write once, run anywhere_.  

Another benefit of being implemented in Java is that muCommander can leverage the large variety of third-party client-side code for different file protocols.  

# Recent changes
Honestly, the [project pulse](https://github.com/mucommander/mucommander/pulse) has not been that great recently. I will touch that later on. Nevertheless, some important changes were done.

## Technical
* Converted the code repository to Git.
* Moved the project to [GitHub](https://github.com/mucommander).
* Replaced the build system with Gradle.
* [Enable compiling the code with Java 9 and Java 10](https://github.com/mucommander/mucommander/pull/158). 
* Manage translations in the [Zanata platform](https://translate.zanata.org/project/view/mucommander).
* Fixed various bugs.

## Communal
* Updated the [website](http://www.mucommander.com).
* Revived the [twitter account](https://twitter.com/mucommander).
* Use [Gitter](https://gitter.im/mucommander/Lobby).

# Challenges
Next, I describe the four major challenges I see at the moment that hinder the progress of the project.  

## Scaling the development model
From time to time we get some really nice contributions as well as issues that users report. However, we currently have a fairly large codebase that becomes hard to maintain by a single maintainer. That causes PRs and issues to occasionally wait relatively long time for getting attention.

## Communicating with the community
The contributions we get and issues that are being filed are a good sign as they show that both developers and end-users are interested in and using muCommander.  

We used to communicate with the developer and user communities by [Google group](https://groups.google.com/d/forum/mucommander-dev), [Forum](http://mu-j.com/mucommander/forums/), and [IRC channel](irc://irc.freenode.net/mucommander). All these are practically abandoned. Seems like both the GitHub issues/PRs and GITTER provide good alternatives for communicating with developers. It may feel like the tools we currently use do not provide a good alternative to the Google group and the Forum for getting feedback and ideas from users though. But on the other hand, that may also be a consequence of the relatively low traffic in general in the project these days.

## Competitive products and projects
There is a large variety of alternative file managers nowadays. Some of them target a specific operating system and thus are sometimes faster and better integrated (the most interesting are probably those that target Mac OS X, that is used by most of our user base). Some are backed up by commercial organizations. Those are typically proprietary products that provide base funtionality for free  and other paid capabilities.  

Another type of alternative products are those that forked from muCommander in the past. Some forks that we have made contact with complained about the development pace of the project that they claim is too slow. That is a shame since migrating features from these forks to muCommander is not always trivial and it could have been much more productive to join forces in a single project.

## Technical debts
Some features that were recently introduced exposed gaps in our platform. Here, I describe two of them:

1. As mentioned before, muCommander was designed to be a **lightweight** file manager, something one can deploy on a minimal USB stick and execute on different machines. That is why we use *proguard* to shrink the jar that is being produced. Nontheless, the size of the produced jar increased due to features such as [supporting vSphere VMs file system](https://yuval.kohavi.info/vsphere/) that brought a dependency of 3.3M with it.
2. When considering something like supporting the qcow2 volumes format (and other virtual disk formats, such as vmdk) using libguestfs, we encounter two issues. First, libguestfs is not available on all operating systems. As mentioned before, muCommander already includes OS-specific things but in this case it would mean having unused dependencies. Second, it requires not only the java dependency (java bindings for libguestfs) but also library code to be installed on the OS (libguestfs). We have no way of specifying such dependencies.


# So what should we do?
Next, I will share some thoughts on how to address the aforementioned challenges.

## Redefine the "mission statement"
The development of muCommander started more than 15 years ago. That is a pretty long time in software terms and so some of the assumptions we began with may not be relevant anymore. For example, the minimal USB stick today contains at least 1G and internet connection is much faster. So considering that muCommander is unlikely to run on devices with limited resources (such as mobile phones), it would probably be alright to produced a larger-size application. As another example, today much more data is stored on cloud services such as dropbox and google drive. Supporting such services may be more important for end-users than things like advanced integrated text editor today.  

So I think this may be the right time to reconsider what is the goal of muCommander - what are its strengths, what should it provide, why should users continue using it and why should developers continue contributing to it.

## Pluggable design
In my opinion, the most important technical change at the moment is introducing a pluggable mechanism. First, it can target only extensions for new file formats and file protocols. Later, it can be extended with other types of extensions.  

Such a pluggalbe mechanism would enable us to:

* Separate out heavy file formats and file protocols from the codebase.
* Install OS-specific extensions only when they are needed.
* Define external dependencies per-extension.

The implementation of a pluggable machnism for muCommander [was already discussed in the past](https://groups.google.com/d/msg/mucommander-dev/-IfxXALXo4U/CJKrhA6A1aYJ). A natural infrastructure to use for this would be OSGI.

## Promoting the project
With an up-to-date "mission statement" and a pluggable mechanism available, we should then promote the project. There are several ways to do this:

* Presenting in a conference, such as FOSDEM or DevConf.cz. 
* Writing an article to a known website, such as [opensource.com](https://opensource.com).
* Defining a list of some desired features (such as PDF viewer, integrating dagger 2, etc) and submitting the project to GSoC (Google Summer of Code).

## Reassess Gitter for user discussions
When things are back on track, we can reassess Gitter as a tool for communicating with end-users. It may be a good idea to schedule a bi-weekly (or a monthly) conference meeting to discuss topics related to the project.