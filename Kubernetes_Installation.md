# Prerequisites:
1. Minimum 2 CPU's with 4Gb Memory is required.
2. Make an entry of each host in /etc/hosts file for name resolution on all kubernetes nodes as below.

```
itadmin@kubemaster:~$ cat /etc/hosts
192.168.123.126 kubemaster
192.168.123.127 kubenode1
192.168.123.128 kubenode2
192.168.123.129 kubenode3
```

3. Make sure kubernetes master and worker nodes are reachable between each other.
4. Kubernetes doesn't support "Swap". Disable Swap on all nodes using below command and also to make it permanent comment out the swap entry in /etc/fstab file.

```
   $ sudo swapoff -a
   $ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
   or
   
   ![image](https://github.com/user-attachments/assets/df752cfb-58ba-44ec-9d6a-c3c0eb77440e)

5. Load the following kernel modules on all the nodes.
```
   $ sudo tee /etc/modules-load.d/containerd.conf <<EOF
   overlay
   br_netfilter
   EOF
   $ sudo modprobe overlay
   $ sudo modprobe br_netfilter
```
6. Set the following Kernel parameters for Kubernetes.
```
   $ sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOT
```

7. Apply sysctl params without reboot
```
   $ sudo sysctl --system
```
8. Internet must be enabled on all nodes, because required packages for kubernetes cluster will be downloaded from official repository.
### 9. Install Containerd Runtime


