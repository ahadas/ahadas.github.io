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

Unfortunately it was not the case in oVirt. The next figure shows the 'self time' of hot-spots in the interaction  with the database in oVirt 3.6 during the time an environment with one host and 6000 VMs was stable for 1 hour. I will elaborate on these number later on, but for now just note that the red color is the overall execution time of DB queries/updates. The more red color we see, the more busy the monitoring is.

![Execution time of DB queries in stable 3.6 environment](http://ahadas.github.io/images/ovirt_scale/3.6-self_time.png)

This work continutes the effort to improve the monitoring aspect in oVirt mentioned in the previous sub-section in order to address this problem. In the next section, I elaborate on the changes we did that lead to the reduced execution times shown in the next figure for the same enviroment in 1 hour (look how much less red color!).

![Execution time of DB queries in stable 4.1 environment](http://ahadas.github.io/images/ovirt_scale/master-self_time.png)

This work:  
* Takes for granted that the monitoring in oVirt hinders its scalability.  
* Does not address hosts monitoring (but only VMs monitoring).  
* Does not refer to other optimizations that do not improve monitoring of a stable system.

# Changes

## Not to process numa nodes data when not needed
We saw that a significant effort was put to process runtime data of numa nodes.  
In terms of CPU time, 8.1% (which is 235 seconds) was wasted on getting all numa nodes of the host from the database and 5.9% (which is 170 seconds) was wasted on getting all numa nodes of VMs from the database. The overall CPU time spent on processing numa node data got up to 14.6%! This finding is similar to what we saw in profiling dump we got for other scaled environment.  
In terms of database interaction, getting this information is relatively cheap (the following are average numbers in micro-seconds):  
* 261 to get numa nodes by host  
* 259 to get assigned numa nodes  
* 255 to get numa node CPU by host  
* 246 to get numa node CPU by VM  
* 242 to get numa nodes by VM

But these queries are called many times so the overall portion of these calls is significant:  
* Getting numa nodes by host - 3% (48,546 msec)
* Getting assigned numa nodes - 3% (48,201 msec)
* Getting numa node CPU by host - 3% (47,569 msec)
* Getting numa node CPU by VM - 2% (45,918 msec)
* Getting numa nodes by VM - 2% (45,041 msec)

The surprising thing was that the host did not report any VM related information about numa nodes! So we changed the analysis of VM's data to skip processing (and fetching from the database) numa related data if no such data is reported for a VM.

## Memoizing host's numa nodes
But we cannot assume that hosts do not report numa nodes data for the VMs. So another improvement was to reduce the number of times that host's level numa nodes data is queried - the query it on per-host basis instead of per-VM. That's ok since this data does not change in the meantime. We used the memoization technique to cache this information during host monitoring cycle.

## Cache VM jobs
Another surprising finding was to see that we put a not negligible effort in processing and updating VM jobs (that is, jobs that represent live snapshot merges) without having a single job like that (the system is stable, remember?).   
It gots up to 3.8% (111 sec) of the overall CPU time and 3% (47,140 msec) of the overall database interactions.  

Therefore, another layer of in-memory management of VM jobs was added. Only when this layers detects that information should be retrieved from the database (not all the data is cached) it access the database.

## Reduce number of updates of dynamic VM data
Despite the use of @UnchangableByVdsm, I discovered that VM dynamic data (that includes for example, its status, ip of client console that is connected to it and so on) is updated. Again, no such update should occur in a stable system... The implications of this issue is significant because this is a per-VM operation so the time it takes is accumulated and in our environment got to 6% (101 sec) of the overall database interactions.

To solve this, VmDynamic was modified. Some of the fields that should not by compared against the reported data were marked with @UnchangableByVdsm and some fields that VmStatistics is a more appropriate place for them were moved.

## Split VM devices monitoring from VMs monitoring
Hosts report the hash of the devices of each VM and the monitoring of the VMs used to compare this hash against the hash that was reported before, and triggers a poll request for full VM data, that contains information of the devices, only when the hash is changed. Not only that the code became more complicaed when it was tangled within other VM analysis computation, but a change in the hash triggered update of the whole VM dynamic data.  

Therefore, we split the VM devices monitoring into a separate module that caches the device hashes and by that reduce even further the number of updates of VM dynamic data. 

## Lighter, dedicated monitoring views
Another observation from analyzing hot spots in the database interactions was that one of the queries we spend a lot of time on is the one for getting the network interface of the monitored VMs. This is relatively cheap query, only 678 micro-sec on average, but it is called per-VM that therefore accumulated to 8% (126 sec) of the overall database interactions.  

The way to improve it was by introducing another query that is based on a lighter view of network interfaces that contains only the information needed for the monitoring.  

This technique was also used to improve the query for all VMs running on a given host. The following output depicts how much lighter is the new view (vms\_monitoring\_view) than the previous view the monitoring used (vms):    

```
engine=> explain analyze select * from vms where run_on_vds='043f5638-f461-4d73-b62d-bc7ccc431429';
 Planning time: 2.947 ms  
 Execution time: 765.774 ms
engine=> explain analyze select * from vms_monitoring_view where run_on_vds='043f5638-f461-4d73-b62d-bc7ccc431429';
 Planning time: 0.387 ms
 Execution time: 275.600 ms
```

This new view is used by the monitoring in oVirt 4.0 but as we will see later on the monitoring in oVirt 4.1 won't use it anymore. Still, this new view is used in several other places instead of the costly 'vms' view.

## In-memory management of VM statistics
The main argument for persisting data into a database is its ability to store information that should be recoverable after restart of the application. However, in oVirt the database is many times used to share data between threads and processes. This badly affects performance.  

VM statistics is a type of data that is not supposed to be recoverable after restart of the application. Thus, one could expect it not to be persisted in the database. But in order share the statistics with thread that queries VMs for clients and with DWH, it used to be persisted.  

As part of this work, VM statistics is no longer persisted into the database. They are now managed in-memory. Threads that queries VMs for clients retrieve it from the memory, and DWH is not supported anymore. By not persisting the statistics, the number of saves to the database it reduced. In our environment it got to 2% (38,669 msec) of the overall database interactions. It also reduce the time it takes to query all VMs for clients.

## Query only VM dynamic data for VM analysis
So 'vms_monitoring_view' turned to be much more efficient than 'vms' view as it returned only statistics, dynamic and static information of the VM (without additional information that is stored in different tables).  

But obviously querying only the dynamic data is much more efficient than using vms\_monitoring\_view:  

```
engine=> explain analyze select * from vms_monitoring_view where run_on_vds='043f5638-f461-4d73-b62d-bc7ccc431429';
 Planning time: 0.405 ms
 Execution time: 275.850 ms
engine=> explain analyze select * from vm_dynamic where run_on_vds='043f5638-f461-4d73-b62d-bc7ccc431429';
 Planning time: 0.109 ms
 Execution time: 2.703 ms
```
So as part of this work, not only that VM statistics are no longer queried from the database but also the static VM data is no longer queried from the database by the monitoring. Each update is done through VmManager that caches only the information needed by the monitoring and the monitoring uses this data instead of queryiing the database. That way, only the dynamic data is queried from the database.

## Eliminate redundant queries by VM pools monitoring
Not directly related to VMs monitoring, VM pool monitoring that is responsible for running prestarted VMs also affects the work done in a stable system. As part of this work, the amount of interactions with the database by VM pool monitoring in system that doesn't contain prestarted VMs was reduced.

# Results
