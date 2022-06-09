---
layout: post
title: Migrating from RHV to Openshift Local
---

Recently I wanted to experience with [Forklift](https://github.com/konveyor/forklift), a project that enables to migrate virtual machines from Red Hat Virtualization (RHV) to Openshift Virtualization. Here, I'll describe how I did this using Openshift Local which can be useful for development or experiments (Openshift Local is not meant for production use).

# Prerequisits

We will need a bare-metal machine or a virtual machine with enough memory and CPU power. I used a virtual machine with 64G of RAM and 18 CPUs but I believe that even 32G of RAM and 10 CPUs would be enough. Additionally, as I won't explain how to deploy RHV, I assume there is a RHV deployment that is accessible from within the machine that Openshift Local would run on.

# Installing Openshift Local

Installing Red Hat Openshift Local is really easy. There is a guide for how to do that on [console.redhat.com](https://console.redhat.com/openshift/create/local). 
Follow the instructions that appear there and make sure it ends successfully and you are able to login to the console with the credentials that appear at the end of the process.

# Adjusting Openshift Local

As we're going to deploy both Openshift Virtualization and the Migration Toolkit for Virtualization on this cluster, we need to override the default settings of the cluster. First, we'll increase the memory to 64G:
```bash
$ crc config set memory 64000
```
Similarly, we'll increase the number of CPUs to 16:
```bash
$ crc config set cpus 16
```

For the previous settings to be applied and since we are now going to extend the virtual disk that is used by the virtual machine that runs the cluster, we need to stop the cluster:
```bash
$ crc stop
```
Once it is stopped, we can extend the aforementioned virtual disk. I extended it by 40G using:
```bash
$ qemu-img resize ~/.crc/machines/crc/crc.qcow2 +40g
```  

Before starting the Openshift cluster again, you can inspect the updated settings with:
```bash
$ crc config view
```
If the settings look alright, start the Openshift cluster with:
```bash
$ crc start
```  

Next. we will extend the filesystem to consume the additional space. In order to do that, we need to log in to the virtual machine using:
```bash
$ ssh -i ~/.crc/machines/crc/id_ecdsa -o StrictHostKeyChecking=no core@<vm ip>
```
You can find the IP address of the virtual machine with:
```bash
$ crc ip
````
Once you're inside the virtual machine, execute the following command:
```bash
$ xfs_growfs /sysroot/
``` 

Congratulations, you now have an Openshift cluster with enough resources to run Openshift Virtualization and Migration Toolkit for Virtualization :)

# Installing Openshift Virtualization

Go to the OperatorHub (Operators -> OperatorHub) and search for 'cnv'. You'll get the Openshift Virtualization operator. Install it and make sure to have both hostpath-provisioner and kubevirt-hyperconverged at the end of the process.  

# Installing Migration Toolkit for Virtualization

Similarly, search for 'mtv' in the OperatorHub and install the Migration Toolkit for Virtualization Operator.

# Setting persistent volumes

Next, we will define a storage class that enables us to provision local persistent volumes (PVs) on the VM that runs the Openshift cluster. This is done by going to Storage->StorageClasses and create a new StorageClass. Give it a name and set the 'Provisioner' field to 'kubevirt.io.hostpath-provisioner'.  

Now we can change some of the built-in PVs to be consumable by this StorageClass. This is done by editing a PV, e.g., with 'oc edit pv pv0009' and set 'accessModes' to 'ReadWriteOnce' only and add 'storageClassName: <storage-class-name>' within the 'spec'.  

You're ready for your first migration!

# Executing a migration plan

In order to initiate a migration you first need to log in to the UI of the Migration Toolkit for Virtualization (MTV). You can find the URL in Networking -> Routes and inspect the 'Location' of the 'virt' route within the openshift-mtv project (namespace). In my case it was 'https://virt-openshift-mtv.apps-crc.testing'. Log in with the same credentials you used for the Openshift console.  

Once you are logged in to the UI of MTV, add a RHV provider under 'Providers'. You should find an Openshift Virtualization provider there as well.  

Then, create mappings - go to the Mappings and define Network and Storage mapping from the source environment (RHV) to the target environment (Openshift Virtualization).  

With that, you can now go to 'Migration Plans' and create a new migration plan. It is fairly simple to do by following the steps in that wizard. Assuming you chose a virtual machine(s) that is installed with a valid guest operating system, the execution of the migration plan would succeed and you'll find the converted virtual machines within the 'Virtualization -> VirtualMachines' view in the Openshift console.
