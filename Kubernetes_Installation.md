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

4. The below command will disable the interactive options while installing anything in server.
```
$ echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
```

5. Kubernetes doesn't support "Swap". Disable Swap on all nodes using below command and also to make it permanent comment out the swap entry in /etc/fstab file.

```
$ sudo swapoff -a
$ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
   or
   
   ![image](https://github.com/user-attachments/assets/df752cfb-58ba-44ec-9d6a-c3c0eb77440e)

6. Load the following kernel modules on all the nodes.
```
$ cat >>/etc/modules-load.d/containerd.conf<<EOF
overlay
br_netfilter
EOF
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```
7. Set the following Kernel parameters for Kubernetes.
```
$ cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
```

8. Apply sysctl params without reboot
```
$ sudo sysctl --system
```
9. Internet must be enabled on all nodes, because required packages for kubernetes cluster will be downloaded from official repository.
### 10. Install Containerd Runtime
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

### 11. Add Apt Repository for Kubernetes
   Kubernetes package is not available in the default Ubuntu 22.04 package repositories. So we need to add Kubernetes repository. run following command to download public signing key,
```
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Next, run following echo command to add Kubernetes apt repository.
```
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
### Note: At the time of writing this guide, Kubernetes v1.31 was available, replace this version with new higher version if available.

### 12. Install Kubectl, Kubeadm and Kubelet
Install Kubernetes components like kubectl, kubelet and Kubeadm utility on all the nodes. Execute following set of commands,
```
$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```
Enable the kubelet service before running kubeadm.
```
$ sudo systemctl enable --now kubelet
```
After here use only the Master Node
### 13. Initialize Kubernetes Cluster
Note: apiserver-advertise-address is your k8s-master ip address
```
$ kubeadm init --apiserver-advertise-address=192.168.123.126 --pod-network-cidr=172.16.0.0/16
```
   After the initialization is complete, you will see a message with instructions on how to join worker nodes to the cluster. Make a note of the kubeadm join command for future reference.

The below commands are to use in regular user. If you are using root user instead of regular user, please skip the below 3 commands.
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Use this below command, if you access via root user.
```
$ export KUBECONFIG=/etc/kubernetes/admin.conf
```
Deploy Calico Network
```
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
```
### 14. Join the Worker Node to Master Node
Copy the __kubeadm token create__ command in the previous step from the master server and run in all the Worker Node.

```
Sample format:
$ kubeadm join 192.168.123.126:6443 --token hp9b0k.1g9tqz8vkf4s5h278ucwf  --discovery-token-ca-cert-hash sha256:32eb67948d72ba99aac9b5bb0305d66a48f43b0798cb2df99c8b1c30708bdc2cased24sf
```
### 15. Verifying the cluster (On k8s-master)
Get Kubernetes Cluster Nodes status
```
$ kubectl get nodes
```
### __Kubernetes Cluster Setup done__

## Retrive the token and ca-cert-token:
Get it from Master Node
```
$ openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1
```
```
$ kubeadm token list
```
After retrieving both token and ca-cert-token:
```
$ kubeadm join 172.16.0.100:6443 --token hp9b0k.1g9tqz8vkf4s5h278ucwf --discovery-token-ca-cert-hash sha256:32eb67948d72ba99aac9b5bb0305d66a48f43b0798cb2df99c8b1c30708bdc2cased24sf
```
