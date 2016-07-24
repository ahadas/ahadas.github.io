---
layout: post
title: Monitoring Improvements in oVirt
---

Recently I've been working on improving the scalability of monitoring in oVirt. That is, how to make oVirt-engine, the central management unit in the oVirt management system, able to process and report changes in a growing number of virtual machines that are running in a data-center. In this post I elaborate on the changes that were done and share some measurements.  

# Background

## Monitoring in oVirt
In short, [oVirt](http://ovirt.org) is an open-source management platform for virtual data-centers. It allows centralized management of a distributed system of virtual machines, compute, storage and networking resources.

In this post the tern *monitoring* refers to the mechanism that oVirt-engine, the central component in the oVirt distributed system, collects runtime data from hosts that virtual machines are running on and reports it to clients. Some examples of such runtime data:  
* Statuses of hosts and VMs  
* Statistics such as memory and cpu consumption  
* Information about devices that are attached to VMs and hosts

## Notable changes in the monitoring unit in the past
### @UnchangableByVdsm
Generally speaking, the monitoring components get runtime date reported from the hosts, compares it with the previously known data and process the changes.  
In order to distinguish dynamic data that is reported by the hosts and dynamic data that is not reported by the hosts, we added in oVirt 3.5 an annotation called UnchangableByVdsm that should be put on every field in VmDynamic class that is not expected to be reported by the hosts. This was supposed to eliminate redundant saves of unchanged runtime data to the database.

### Split hosts-monitoring and VMs-monitoring
Previously, before monitoring a host we locked the host and released it only after the host-related information and all information of VMs running on the host was processed. As a result, when doing an operation on a running VM, the host that the VM runs on was locked.  
A major change in oVirt 3.5 was a refactoring the monitoring to use per-VM lock instead of the host lock while processing VM runtime data. That reduced the time that both monitoring and command threads are locked.

### Introduction of events based protocol with the hosts
A highly desirable enhancement came in oVirt 3.6 in which changes in VM runtime data are reported as events rather than by polling. In a typical data-center many of the VMs are 'stable', i.e. their status does not change much. In such environment, this change reduces the number of data sent on the wire and reduces the unnecessary processing in oVirt-engine.  
Note that not all monitoring cycles were replaces with events: on every 15 seconds (by default), oVirt-engine still polls the statuses of all VMs, including their statistics. These cycles are called 'statistics cycles'.

## Scope of this work
An indication that monitoring is inefficient is when it does too many things while the system is stable (I prefer the term 'stable' over 'idle' since the virtual machines could actually be in use). For example, when virtual machnines don't change (no operation is done on these VMs and nothing in their environment is changed), one could expect the monitoring not to do anything except for processing and persisting statistics data (that is likely to change frequently).  

Unfortunately it was not the case in oVirt. Figure 1 shows the 'self time' of hot-spots in the interaction  with the database in oVirt 3.6 during the time an environment with one host and 6000 VMs was stable. I will elaborate on these number later on, but for now just note that the red color is the overall execution time of DB queries/updates. The more red color we see, the more busy the monitoring is.

![Figure 1 - Execution time of DB queries in stable 3.6 environment](http://ahadas.github.io/images/ovirt_scale/3.6-self_time.png)

This work continutes the effort to improve the monitoring aspect in oVirt mentioned in the previous sub-section in order to address this problem. In the next section, I elaborate on the changes we did that lead to the reduced execution times shown in Figure 2 (look how much less red color!).

![Figure 2 - Execution time of DB queries in stable 4.1 environment](http://ahadas.github.io/images/ovirt_scale/master-self_time.png)

This work:  
* Takes for granted that the monitoring in oVirt hinders its scalability.  
* Does not address hosts monitoring (but only VMs monitoring).  
* Does not refer to other optimizations that do not improve monitoring of a stable system.