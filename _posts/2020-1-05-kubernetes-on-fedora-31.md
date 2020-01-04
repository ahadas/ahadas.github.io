---
layout: post
title: Plain Kubernetes Cluster on Fedora 31
---

In this post I describe how to run a plain (vanilla) Kubernetes cluster that spreads across physical machines installed with Fedora 31.  

Various projects, like the [Kubevirt project](https://kubevirt.io/) that I've been involved in recently, introduce an automated way for initiating a Kubernetes cluster. In Kubevirt, for instance, there is a framework named [kubevirtci](https://github.com/kubevirt/kubevirtci) that enables one to quickly spin up and destroy Kubernetes clusters for testing. The kubevirtci framework is composed of two parts. The first part initiates a cluster. The second part deploys Kubevirt on top of that cluster. I can say kubevirtci served me well during the time I developed Kubevirt and the scripts and configurations used by kubevirtci may be used by others to easily run Kubernetes/Openshift clusters that spread across multiple virtual machines within a single physical machine.  

However, as part of a new project I started to work on I needed to run Kubernetes clusters across distributed virtual machines (that can be considered physical machines on the same network) for which kubevirtci does not fit, and so following are the steps I've made that I share in the hope others that attempt to achieve the same thing would find it useful.  

First, we need to installl docker as explained [in this guide](https://linuxconfig.org/how-to-install-docker-on-fedora-31).

Then change the `cgroup-driver` of docker to be systemd by extending `ExecStart` in `/etc/systemd/system/multi-user.target.wants/docker.service` with:
```
--exec-opt native.cgroupdriver=systemd
```

The next step is following [this guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), and specifically:

Add Kubernetes repository:
```bash
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

Disable SELinux:  
```bash
$ sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```  

Disable swap in `/etc/fstab`.

Disable the firewall:
```bash
$ systemctl disable firewalld
```  

Install Kubernetes packages:
```bash
$ yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable kubelet:
```bash
$ systemctl enable --now kubelet
```

Ensure `net.bridge.bridge-nf-call-iptables` is set to 1 in your sysctl config:
```bash
$ sysctl net.bridge.bridge-nf-call-iptables=1
```

Install `kubernetes-cni`:
```bash
$ dnf install kubernetes-cni
```  

Reboot the machine:
```bash
$ shutdown -r now
```

On the master node, run:
```bash
$ kubeadm init
```

Then deploy `weave-net`:
```bash
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.32.0.0/16"
```

On the worker nodes run `kubeadm join` with the token that was returned by `kubeadm init` on the master node. This, as well as `kubeadm init` on the master node, can be reverted back with `kubeadm reset`.

To check your cluster is up and running, inspect the nodes by running on the master node:
```bash
$ kubectl get nodes
```

And the pods:  
```bash
$ kubectl get pods -n kube-system
```

To inspect the logs of kubelet, run:
```bash
$ journalctl -ur kubelet.service
```
