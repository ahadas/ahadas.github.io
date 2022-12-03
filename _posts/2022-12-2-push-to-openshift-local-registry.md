---
layout: post
title: Push Custom Images to Openshift Local
---

This post describes the next step in our journey to deploy MTV (Migration Toolkit for Virtualization) on Openshift Local: deploying a custom image to Openshift Local without going through an external registry like quay.io

# Prerequisits

Set up an Openshift Local cluster and deploy MTV on it as described [here](http://ahadas.com/mtv-on-openshift-local/) (note that you should also make the PVs accessible to the pods for being able to start VMs).  

Clone [the repository of Forklift](https://github.com/kubev2v/forklift) and make sure you are able to build the image of forklift-controller with `make build-controller`.

# Expose the image-registry

Do the following steps that are taken from the [Openshift documentation](https://docs.openshift.com/container-platform/4.11/registry/securing-exposing-registry.html):  

```bash
$ HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

```raw
$ oc get secret -n openshift-ingress  router-certs-default -o go-template='{{index .data "tls.crt"}}' | base64 -d | sudo tee /etc/pki/ca-trust/source/anchors/${HOST}.crt  > /dev/null
```

```bash
$ sudo update-ca-trust enable
```

```bash
$ podman login -u kubeadmin -p $(oc whoami -t) $HOST
```

Note that the last step above is different than what is written in the above mentioned Openshift documentation, this command shouldn't be executed with `sudo`.

# Push an image to the exposed registry

Tag an image with $HOST as the registry and push it, for example:
```bash
$ podman tag <image-id> $HOST/openshift/forklift-controller:devel
$ podman push $HOST/openshift/forklift-controller:devel
```

Specifically, for forklift-controller this can be achieved with:
```bash
$ export IMG=$HOST/openshift/forklift-controller:devel
$ make push-controller
``` 

# Redirect the pod to use the image from the cluster's image registry
This part depends on the way the application is deployed, here I'll describe a common practice for applications that are deployed using operators in which the image from the internal/cluster's image registry is injected by the operator.  

First, identify the ClusterServiceVersion instance in the relevant namespace (in my case it was called `mtv-operator.v2.3.1` in the `openshift-mtv` namespace).  

Then, edit it and specify the image that was pushed to the internal registry, in my case it was done by editing `mtv-operator.v2.3.1`:
```bash
$ oc edit csv -n openshift-mtv mtv-operator.v2.3.1
```
And setting the value of RELATED_IMAGE_CONTROLLER to: `image-registry.openshift-image-registry.svc:5000/openshift/forklift-controller:devel`

Finally, delete the pod, in my case forklift-controller, so another one would start with the new image from the internal registry.
