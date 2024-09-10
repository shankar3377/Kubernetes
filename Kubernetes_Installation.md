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
   In this guide, we are using containerd runtime for our Kubernetes cluster. So, to install containerd, first install its dependencies.
```
$ sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

Enable the Docker Reporsitory.
```
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
Now, run following apt command to install containerd
```
$ sudo apt update
$ sudo apt install -y containerd.io
```
Configure containerd so that it starts using systemd as cgroup.
```
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Restart and enable containerd service.
```
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```

### 9. Add Apt Repository for Kubernetes
   Kubernetes package is not available in the default Ubuntu 22.04 package repositories. So we need to add Kubernetes repository. run following command to download public signing key,
```
curlable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
