---
layout: post
title: Improved Virtual Machines Monitoring in oVirt
---

Recently I've been working on improving the scalability of monitoring in oVirt. That is, how to make oVirt-engine, the central management unit in the oVirt management system, able to process and report changes in a growing number of virtual machines that are running in a data-center. In this post I elaborate on the changes that were done and share some measurements.  

# Background

## oVirt
In short, [oVirt](http://ovirt.org) is an open-source management platform for virtual data-centers. It allows centralized management of a distributed system of virtual machines, compute, storage and networking resources.

*Virtual machines* are running on different *hosts* on top of the KVM hypervisor. oVirt-engine is a central component that is responsible for executing requests it gets from clients and monitor virtual machines, hosts and other resources and report the up-to-date status of the environment back to clients.

## Previous notable changes in the monitoring
### @UnchangableByVdsm
Generally speaking, the monitoring components get runtime date reported from the hosts, compare it with the previously known data and process the changes.  
In order to distinguish dynamic data that is reported by the hosts and dynamic data that is not reported, we added an annotation called UnchangableByVdsm that should be put on every field in VmDynamic class that is not expected to be reported by the hosts. This was supposed to eliminate redundant processing in the monitoring of unchanged runtime data.


### Split hosts-monitoring and VMs-monitoring
Previously, before monitoring a host we locked the host and released it only after the host-related information and all information of VMs running on the host was processed. When doing an operation on a running VM, the host that the VM runs on was locked.  
A major change in oVirt 3.5 was to refactor the monitoring to use per-VM lock instead of the host lock. That reduced the time that both monitoring and command threads are locked.

### Introduction of events based protocol with the hosts
A highly desirable enhancement came in oVirt 3.6 in which some of the VMs reports are reported as events rather than polling. In a typical data-center, many of the VMs are 'stable', i.e. their status does not change much. In such environment, this change reduces the number of data sent on the wire and reduces the unnecessary processing in oVirt-engine.  
Note that not all monitoring cycles were replaces with events: on every 15 seconds (by default), oVirt-engine polls the statuses of all VMs, including their statistics. These cycles are called 'statistics cycles'.

## Scope of this work
This work continutes the work mentioned above. While changes were done in different areas, this work concentrates only on VMs monitoring.  

An indication that monitoring is inefficient is to see that it does non negligible work while the system is stable (I prefer the term 'stable' over 'idle' since the virtual machines could actually be in use). When virtual machnines don't change, one could expect the monitoring not to do much. That is, not to get any event and process only statistics data (that is likely to change frequently) in this period of time.  

However, in practice this was not the case in oVirt. Figure 1 shows the 'self time' of hot-spots during the time an environment with one host and 6000 VMs was stable. I will elaborate on these number later on, but for now just note that the red color is the overall execution time of DB queries. The more red color we see, the more busy the monitoring is.

![Figure 1 - Execution time of DB queries in stable 3.6 environment](images/ovirt_scale/3.6-self_time.png)

Next we elaborate on the changes that lead to the reduced execution times shown in Figure 2 (look how much less red color!).

![Figure 2 - Execution time of DB queries in stable 4.1 environment](images/ovirt_scale/master-self_time.png)
