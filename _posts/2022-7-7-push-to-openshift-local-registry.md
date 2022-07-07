---
layout: post
title: Push Custom Images to Openshift Local
---

This post describes the next step in our journey to deploy MTV (Migration Toolkit for Virtualization) on Openshift Local: deploying a custom image to Openshift Local without going through an external registry like quay.io

# Prerequisits

Set up an Openshift Local cluster and deploy MTV on it as described [here](http://ahadas.com/mtv-on-openshift-local/) (note that you should also make the PVs accessible to the pods for being able to start VMs).  

Clone a repo of Forklift, like [Forklift Controller](https://github.com/konveyor/forklift-controller) and make sure you are able to build (`podman build .`) and tag an image (`podman tag <image-id> <tag-id>`).  

# Expose image-registry

The image registry service is called image-registry and it resides in the openshift-image-registry namespace. We'll start by defining a port-forward in order to expose it outside of the cluster:  
```bash
$ oc port-forward --address 0.0.0.0 -n openshift-image-registry svc/image-registry 5000:5000 &
```

# Accessing image-registry from outside

Next we'll update /etc/hosts on the outer machine:  
```bash
$ echo "<crc-ip> image-registry.openshift-image-registry.svc" >> /etc/hosts
```

The certificate that is used from the Openshift Local VM to access the image-registry can be found on: /etc/docker/certs.d/image-registry.openshift-image-registry.svc:5000/.  
Since we prefer to use podman on our (outer) machine the certificates are placed at a different place: /etc/containers/certs.d/. Make a folder `image-registry.openshift-image-registry.svc:5000` there and copy the ca.crt from the aforementioned folder in the Openshift Local VM to this folder.  

With that, we should now be able to login to the registry from the outer machine with:
```bach
$ podman login -u kubeadmin -p $(oc whoami -t) image-registry.openshift-image-registry.svc:5000
```

Note that you should be logged in with `oc` before this with the credentials you got during the installation of Openshift Local.  

# Pushing a custom image
First, tag your image, e.g.:
```bash
$ podman tag 11873350495c image-registry.openshift-image-registry.svc:5000/openshift/forklift
```

Second, push that image:
```bash
$ podman push image-registry.openshift-image-registry.svc:5000/openshift/forklift
```

Since we didn't specify a tag, the tag defaults to `latest`.

# Using a custom image for forklift-controller
This part is application-specific but this is a common pattern to apps that are deployed with operators. We will change the image that is injected to the operator.  

First, identify the ClusterServiceVersion in the relevant namespace (in my case it was called mtv-operator.v2.3.1 in the openshift-mtv namespace).  

Then, edit it and specify the image that was pushed, in my case it was done by editing mtv-operator.v2.3.1:
```bash
$ oc edit csv -n openshift-mtv mtv-operator.v2.3.1
```
And setting the value of RELATED_IMAGE_CONTROLLER to: image-registry.openshift-image-registry.svc:5000/openshift/forklift:latest

Then delete the operator deployment and reprovision the Forklift Controller, which will now use the new image from the internal registry.
