---
layout: post
title: oVirt-Specific Aspect Languages
---

This post demonstrates the development and use of *Application-Specific Aspect Languages* through an example of such language for synchronization in [oVirt](/about/#ovirt).  

# Background

## Application-Specific Aspect Languages
*Aspect-oriented programming (AOP) languages* provide a modularization layer for *crosscutting-concerns*. These are requirements whose implementation does not conform the dominant design of the software at hand. AOP promotes the implementation of a crosscutting-concern within a modularization unit called *aspect* that would be woven into the *base code* instead of having it scattered and tangled across the *base code* along with the core business logic of the application.  

*Domain-specific languages (DSLs)* are programming languages that are tailored to problems of a specific domain. By being more restrictive and allowing programmers to use notions and abstractions that are closer to the problem at hand, DSL is typically easier to program with than a general-purpose language. *Language workbenches* provide supportive tools for developing DSLs and programming with DSLs, thus facilitate the *language-oriented programming (LOP)* paradigm that promotes the development and use of DSLs as part of software development process.

*Domain-specific aspect languages (DSALs)* are programming languages that are both aspect-oriented and domain-specific. DSAL allows to resolve crosscutting-concern of a particular domain by implementing it in its own module and expressing it with a domain-specific syntax.  

*Application-Specific aspect languages* (or *disposable aspect languages*) are a sub-group of DSALs that are tailored to a given instance of problem. They can be easier to design, to implement and to program with than DSALs at the expense of being less reusable.

## Synchronization in oVirt
Some of the operations that are supported by oVirt cannot be executed simultaneously and it is one of the responsibilities of ovirt-engine, the central component in oVirt that clients interact with, to prevent such conflicts from happening.

### Evolution of a Crosscutting-Concern
Synchronization is considered to be a classic example of a crosscutting-concern, and the synchronization needed in oVirt is no exception.  

The core responsibility of ovirt-engine is to execute operations it gets from clients. Therefore, it only makes sense that its design is based on the *[Command pattern](https://en.wikipedia.org/wiki/Command_pattern)*. Encapsulating each operation as a command class makes the software easier to maintain and having common root class for all commands allows to treat them uniformly.  

However, synchronization handling does not fit well to this design. In order to be able to synchronize we need some per-command configuration and some handling that is common for all commands. The per-command configuration is scattered across most of the command classes and the common handling resides in the common root class called CommandBase where it is tangled with other code that is common for all the commands.  

At the beginning, the locking mechanism in oVirt on which the synchronization handling is based was relatively simple. Each command used in-memory lock for a short period of time in which the relevant entities were locked in the database. The locks in the database were cleared when the command is finished.  

But this design did not fit for other requirements that were added as the application evolved. For example, addition of read-locks in the database (in order to support [multiple VM creations from the same template](https://bugzilla.redhat.com/show_bug.cgi?id=815642)) required to change many entities in the database. As a result, the locking mechanism was enhanced with different constructs for different flows that made it more complex.  

### Internal DSL
The specification of the lock properties is one of the things that became more complex to define. [Bugzilla 995836](https://bugzilla.redhat.com/show_bug.cgi?id=995836) describes an attempt to improve it by replacing the previously used Java annotation with an *[Internal DSL](http://martinfowler.com/books/dsl.html)* (also known as *[Fluent API](https://en.wikipedia.org/wiki/Fluent_interface)*). An example of lock properties specification with the internal DSL:

```java
protected LockProperties applyLockProperties(LockProperties lockProperties) {
    return lockProperties.withScope(Scope.Execution).withWait(true);
}
```

While the benefit of having internal DSL vs. use of Java annotations for defining the desired lock properties is debatable, it is clear that the fundamental problems of scattered and tangled code remain. The scattered lock configurations leads to low traceability that reduces productivity and produces bugs, such as [bugzilla 1251956](https://bugzilla.redhat.com/show_bug.cgi?id=1251956). In addition, CommandBase is hard to maintain because of high tangling.

# oVirt-Specific Aspect Language
Obviously, (either external or internal) DSL is not the right solution for the described synchronization concern. DSL can simplify the expression of part of the locks configuration, but it cannot improve the modularization of the scattered and tangled code.  

The synchronization concern can be modularized with a general-purpose aspect language, such as [AspectJ](https://eclipse.org/aspectj). However, since AspectJ is an extension to Java, most of oVirt developers would need to know additional programming language which puts the cost-effectiveness into question.  

## ovirt-sync: Aspect Language for Synchronization in oVirt
Application-specific aspect language can improve the modularization of crosscutting-concern (unlike DSL) and improve the cost-effectiveness of using the language (vs. general-purpose aspect language.  

The following video shows part of the configuration of an application-specific aspect language for the synchronization problem in oVirt, called ovirt-sync, using the [Xtext](https://eclipse.org/Xtext) language workbench. The grammar of the language is defined in Locks.xtext file and the transformation of an aspect written in ovirt-sync into an aspect in AspectJ is defined in LocksGenerator.xtend. The last part of the video shows an aspect written in ovirt-sync (locks.ovirt) and the AspectJ aspect it generates (Locks.aj).  

<a href="http://www.youtube.com/watch?feature=player_embedded&v=uj80yWutQak" target="_blank"><img src="http://img.youtube.com/vi/uj80yWutQak/0.jpg" alt="Developing a DSAL for synchronization in oVirt" width="240" height="180" border="10" /></a>

The next video demonstrates the development of the synchronization needed for one command in oVirt, MigrateVmCommand, in ovirt-sync. Note the editing tools that are provided such as auto-completion, text-highlighting and syntax-error checking along with aspect-specific development tools like AJDT markers (shown next to line 1,2 that indicate this code affect certain points in the base code. Similar markers are also shown next to the affected places in the base code). The last part shows the generated AspectJ aspect.  

<a href="http://www.youtube.com/watch?feature=player_embedded&v=PTy9rYDQSo4" target="_blank"><img src="http://img.youtube.com/vi/PTy9rYDQSo4/0.jpg" alt="Resolving synchronization in oVirt with DSAL" width="240" height="180" border="10" /></a>

## Is it Practical?
Obviously, programming with them is easier than with general purpose languages thanks to the declarative and restrictive syntax along with tool support, as demonstrated by the second video (this video demonstrates programming with ovirt-sync in Eclipse but IDE plugin can also be generated for IntelliJ). The question to be asked is whether they are cost-effectiveness considering their development cost.

While it might not be reflected by the first video, as it only shows the products of the language development, the effort needed to develop the language was relatively low thanks to the supportive development tools provided by Xtext. For one that knows AspectJ and the Xtext language workbench, the effort of language development of a language similar to ovirt-sync is estimated in couple of hours. Note that only some of the developers in the project need to have the required knowledge for developing languages and most of the developers enjoy programming with a much simpler (configuration-like) language.

The locks for three commands in oVirt (namely, MigrateVmCommand, AddDiskCommand and ExportVmTemplateCommand) were defined in ovirt-sync as well as auditing and permission checks that were defined in two other application-specific aspect languages. Clearly, programming in these languages significantly reduces the amount of scattered code within the command classes and the amount of tangled code within CommandBase.

# Summary and Future Work
The development of application-specific aspect languages for different crosscutting-concerns found in oVirt and programming with them show the power of these languages in improving software modularity and their improved cost-effectiveness compared to ordinary domain-specific aspect languages.  

I obtained the first place in the ACM student research competition at [Modularity'16](http://2016.modularity.info/) where this work was presented. Different aspects of this work were presented in three workshops at Modularity'16. More information and resources are available [here](/about#presentations).    

The AspectJ compiler (ajc) was modified to support these languages. The compiler development for better support for *Language-Oriented Modularization (LOM)*, a methodology that promotes the development and use of DSALs as part of the software modularization process, is a work-in-progress.
