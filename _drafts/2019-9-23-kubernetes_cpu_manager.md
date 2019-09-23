The assignments done by CPU manager:  
/var/lib/kubelet/cpu_manager_state

Building RPM of upstream k8s:  
bazel build rpms  
version: -v=8  
sudo kubeadm init --kubernetes-version=latest-1 --system-reserved=cpu=500m,memory=400Mi  

Logs:  
journalctl -u kubelet.service (-r : reversed, -f : following)  

Set 'static' policy:  
/var/lib/kubelet/config.yaml  

Set reserved-resources:  
/etc/sysconfig/kubelet  
