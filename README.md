This README document provides instructions on manually setting up a Kubernetes cluster. The article will also cover how to use MetalLB for load balancing and the Kubernetes Dashboard for monitoring and managing the cluster. The goal is to give readers a basic understanding of Kubernetes and the tools to create their playground for testing and simulating scenarios.


Create a Kubernetes cluster with [@Rocky Linux minimum](https://rockylinux.org/download) installation on the vsphere environment. 

This cluster consists of a single master node and two worker nodes. we will use [@kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) to initialize the cluster. 

At a later phase, let's add the [@metallb](https://metallb.universe.tf/installation/) for the load balancer IP range and [@dashboard](https://github.com/kubernetes/dashboard) for GUI. 

Basic idea here to map a fundamental container components like compute, network and storage (at later phase) to together. 

Compute - CRI (Container Runtime Interface) [@containerd](https://github.com/containerd/containerd)

Network - CNI (Container Network Interface) [@calico](https://github.com/projectcalico/calico)

Storage - CSI (Container Storage Interface) [@Dell PowerStore](https://github.com/dell/csi-powerstore) ## Futuer Blog




## Network Diagram

![Diagram1](https://github.com/cha2ranga/k8s-installation/blob/main/images/diagram1.jpg)


## VMs Installation 

Setup VMs with minimum installation (2vCPU, 4GB Memory, 30GB Disks)
Once installation is finishd, setup static IPs (preffered) for all threee VMs. 
Then upgrade the OS to latest patch versions for all three VMs. 
```bash
  yum install -y wget
  yum update -y && reboot
```
Once nodes are back in online, setup /etc/hosts file as follows. If you have proper DNS in the environment feel free to follow 

Since we are following non-production deployment add the host file entries. 
```bash
 cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.27.3.101    master
172.27.3.102    worker1
172.27.3.103    worker2
```

## Configure passwordless SSH
Configure ssh-key files
### on master 
```bash
ssh-keygen
```
Once key is ready you can copy the key to worker1 and worker2
```bash
ssh-copy-id root@worker1
ssh-copy-id root@worker2
```
Now from master node you will be able to ssh into both worker nodes without password. 




## RUN below 04 stpes into all three VMs

## 01 VM prepreration for k8s installation

There are multiple parameter need to be configured before we start k8s installation. Here we are going to use [@containerd](https://github.com/containerd/containerd) as a CRI.

Rest of the prerequsites likes, swap configuration, addtional packages, firewall, addtional kernel modules will be configured using basic bash scripts. 

If you want to adjust specific version of the k8s installation, you can manually edit the scripts. Define specific version number. 
Otherwise this installation will follow lates installation. 

Create a directory and download scripts

```bash
cd ~ && mkdir k8s_installation && cd k8s_installation
```
```bash
wget https://raw.githubusercontent.com/cha2ranga/k8s-installation/main/scripts/1_k8s_install_part1.bash
wget https://raw.githubusercontent.com/cha2ranga/k8s-installation/main/scripts/2_k8s_install_part2.bash
```
```bash
chmode +x 1_k8s_install_part1.bash
chmode +x 2_k8s_install_part2.bash
```


## 02 - Run Script 1_k8s_install_part1.bash
Script1 will adjust/install firewall, addtional kernel moduels, Container run time [@containerd](https://github.com/containerd/containerd) etc. 
```bash
./1_k8s_install_part1.bash
```
Script will automatically adjust the /etc/containerd/config.toml config. 
Change the value of cgroup driver "SystemdCgroup = false" to "SystemdCgroup = true". This will enable the systemd cgroup driver for the containerd container runtime.

![containerd configuration](https://github.com/cha2ranga/k8s-installation/blob/main/images/containerd1.jpg)


## 03 - Run Script 2_k8s_install_part2.bash
Scrip2 will install/configure kubelet, kubeadm, kubectl and addtional packages like multipath. 
```bash
./2_k8s_install_part2.bash
```

## 04 - Configure socket path for containerd
once you properly configure container socket path, you will be able to list down containers in the individual node. 

```bash
crictl config runtime-endpoint unix:///run/containerd/containerd.sock
```

## Now prerequsites are completed. 
Let's create a kubernetes cluster using [@kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) tool.


## Use KUBEADM to create kubernetes Cluster
## Run below commands from your master node

we are going to use following ip cidr for container network
10.244.0.0/16

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

After kubeadm initialized the master nodes, you will see base containers are up and running in the master node. 
```bash
crictl ps
```

Now you can start enabling network communication for your container network plugin. Here we are using [@calico manifest](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises) file. 

Download the yaml file

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

then apply CNI
```bash
kubectl apply -f calico.yaml
```



## Join Worker Nodes to cluster
Get the token and use this token to join rest of the worker nodes to cluster. 

```bash
kubeadm join 172.xx.xx.xx:6443 --token z1v1jp.pmxxxxxxxxxxxcmqt3 \
        --discovery-token-ca-cert-hash sha256:0924f8614xxxxxxxxxxxxxxxxxx56a52a1bf5a2b1fbc575e8
```

In case if you missed that token, you can re-create the token using follwoing command
```bash
kubeadm token create --print-join-command
```
 
Go to each worker node and join them to cluster


## Metallb Installation

Here we are going to use Metallb as a L2 mode. 
metallb iprange 172.27.1.60-172.27.1.69

Refer to this URL for [@metallb manifest](https://metallb.universe.tf/installation/) installation. 

apply the manifest by,
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```
Manifest will create a "metallb-system" namespace

Now you can refer to [@L2 configuration](https://metallb.universe.tf/configuration/_advanced_ipaddresspool_config/)

First, create an ip address pool for your load balancer. Add the below content to your l2ip_pool.yml file
```bash
touch l2ip_pool.yml
```
```bash
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.27.1.60-172.27.1.69
```

then create IP pool
```bash
kubectl apply -f l2ip_pool.yml

```
output:

ipaddresspool.metallb.io/first-pool created


Verify load balancer IP pool

```bash
k -n metallb-system get ipaddresspools.metallb.io
```
output:

NAME         AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES

first-pool   true          false             ["172.27.1.60-172.27.1.69"]

Let's advertise the ip pool. first create "l2adv.yml"

```bash
touch l2adv.yml
```
```bash
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool

```

Let's advertise IP range
```bash
kubectl create -f l2adv.yml
```
output:

l2advertisement.metallb.io/example created

verify advertised ip pool
```bash
kubectl -n metallb-system get l2advertisements.metallb.io
```

output:

NAME      IPADDRESSPOOLS   IPADDRESSPOOL SELECTORS   INTERFACES

example   ["first-pool"]


Now you can change your sample deployment of the web application to load balancer
```bash
kubectl expose deployment web --port=80 --name=websvc --type=LoadBalancer
```

verify,

k get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE

kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        19d

websvc       LoadBalancer   10.111.84.155   172.27.1.60   80:32179/TCP   5s



## Metric Server Installation

The Kubernetes Metrics Server is a cluster-wide aggregator of resource usage data.

Even though installation is simple, it usually will not work due to certificate and other configuration errors. There are additional parameters that need to add based on documentation.

Here are the common challenges with [@Metrics-server](https://github.com/kubernetes-sigs/metrics-server/blob/master/KNOWN_ISSUES.md#kubelet-doesnt-report-pod-metrics) installation.

1. create config map for "front-proxy" certificate
```bash
kubectl -n kube-system create configmap front-proxy-ca --from-file=front-proxy-ca.crt=/etc/kubernetes/pki/front-proxy-ca.crt -o yaml | kubectl -nkube-system replace configmap front-proxy-ca -f -
```

output:
configmap/front-proxy-ca replaced

verify,

```bash
kubectl -n kube-system get cm
```

2. Now you can apply the metrics-server manifest file
```bash
wget https://raw.githubusercontent.com/cha2ranga/k8s-installation/main/metrics-server/components_custom.yaml
```

```bash
kubectl apply -f components_custom.yaml
```

3. now, you can run the following commands to verify the metrics server. Note: wait at least for 30 seconds 
```bash
kubectl top nodes
```
```bash
kubectl top pods
```



