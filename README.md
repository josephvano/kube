Example of setting up a local environment with Kubernetes cluster. Lots of the examples are from the book manning book [Kubernetes in Action](https://www.manning.com/books/kubernetes-in-action).

This requires you have [Vagrant](https://www.vagrantup.com/) installed on your computer.

# Vagrant installation

Follow the steps located on the [vagrant site](https://www.vagrantup.com/downloads.html) to get it up and running. 

Once installed on your computer, modify the **Vagrantfile** in this repository:
* update the `vm.network "public_network", bridge: "en0: [YOUR NETWORK INTERFACE]"` bridge value section in each vm configuration to your specific network interface
* tweak the cpus and memory for your situation, defaults are fine also

If you are unsure what the full name of network interface you should assign, 
you can remove the value and it will prompt you `vagrant up`. 
From there it will display the full name of the interface for you to select.


Without specifying the network interface:
```ruby
node.vm.network "public_network"
```

After modifying the `Vagrantfile`, run the following command witin the same directory
of the file to provision all the required machines to get a cluster going.

```bash
# vagrant up
```

# Local Cluster Installation
After getting all the nodes up and running, 
you must install the container (Docker) runtime and kubernetes tools on each node.

Follow these next two sections to get Docker and Kubernetes installed on each of the nodes.

## Docker Installation
This references a lot from the official site for [installing docker on CentOS 7](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository)

Before you install Docker Engine - Community for the first time on a new host machine, 
you need to set up the Docker repository. Afterward, you can install 
and update Docker from the repository.

Remove previous versions, if applicable
```bash
# sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

Using the repository method on CentOS 7

```
# sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
# sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# sudo yum -y install docker-ce docker-ce-cli containerd.io
# sudo systemctl enable docker && sudo systemctl start docker
```

Verify docker installation
```
docker --version
```

## Kubernetes Installation
### Requirements
Master node must have the following requirements
* 2 cpus
* 2GB ram
* OS
** Centos 7+
** Ubuntu 16+
* Full network connectivity between machines
* Unique hostname
* Disable swap for kubelet to work
* Specific [ports open](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

### Disable Security
```bash
# sudo setenforce 0
# sudo systemctl disable firewalld && sudo systemctl stop firewalld
```

But this only disables it temporarily (until the next reboot). 
To disable it permanently, edit the `/etc/selinux/config` file 
and change the SELINUX=enforcing line to **SELINUX=permissive**.

### Repository File

Create a new repository file named **kubernetes.repo** in `/etc/yum.repos.d/`

```
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

### Install packages
```bash
# sudo yum install -y kubelet kubeadm kubectl kubernetes-cni
# sudo systemctl enable kubelet && sudo systemctl start kubelet
```

* kubelet—The Kubernetes node agent, which will run everything for you
* kubeadm—A tool for deploying multi-node Kubernetes clusters
* kubectl—The command line tool for interacting with Kubernetes
* kubernetes-cni—The Kubernetes Container Networking Interface

Enabling the net.bridge.bridge-nf-call-iptables kernel option

I’ve noticed that something disables the bridge-nf-call-iptables kernel parameter, which is required for Kubernetes services to operate properly. To rectify the problem, you need to run the following two commands:

```bash
# sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
# echo "net.bridge.bridge-nf-call-iptables=1" > /etc/sysctl.d/k8s.conf
```

### Disable Swap

```bash
# sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Initialize Master

After installation is done on all the nodes with docker and 
kubernetes packages, you can setup the master node.

Go ahead and ssh into the master node
```bash
# vagrant ssh master
```

When initializing the master node, we supply two options so it binds to the 
correct IP and creates a pod network for the right pod networking add on

```bash
# kubeadm init --apiserver-advertise-address [MASTER_NODE_IP] --pod-network-cidr=192.168.0.0/16
```

Sample output message after initialization. Take note of the `kubeadm join` message
since this is what you will run on each of your nodes to join the cluster.
> 
> Your Kubernetes control-plane has initialized successfully!
> 
> To start using your cluster, you need to run the following as a regular user:
> 
>   mkdir -p $HOME/.kube
>   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>   sudo chown $(id -u):$(id -g) $HOME/.kube/config
> 
> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>   https://kubernetes.io/docs/concepts/cluster-administration/addons/
> 
> Then you can join any number of worker nodes by running the following on each as root:
> 
> kubeadm join [MASTER_NODE_IP]:6443 --v=5 --token [TOKEN] \
>    --discovery-token-ca-cert-hash [SHA]
>

### Setup kubectl on master node
Run the following command on your master node to control the cluster.
You can also add this to your shell profile.

```bash
 # export KUBECONFIG=/etc/kubernetes/admin.conf
 ```

### Setup pod networking 

It's required to install a pod network so that pods can talk to each other across nodes
There are lots of [options for setting up pod networking](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network). 

The following two are ones that I used that worked somewhat on the machines I provisioned
* Weave net 
* Calico

Run **only one of the following** commands on the master node to setup pod networking, either calico or weave.

#### Calico
Calico seemed to work better. 
You have to make sure you initialize the master with the pod cidr block
https://docs.projectcalico.org/getting-started/kubernetes/quickstart

```bash
# sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

```

#### Weave Net
```bash
# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## Join Worker Nodes

After master is initialized and the pod networking is setup, you can have
each worker node join the cluster.

Go ahead and ssh into each worker node and run the `kubadm join` command
you copied from the initialization of the master node. Ensure the ip of
the master node is correct and accessible to the worker nodes.

## Hosts setup
These are just some steps to make it easier to navigate through your cluster. 
This is not required for setup.

### hostnames
To easily access the nodes, you can update the host files with the hostnames and ips 
to the other nodes. Just ssh into each node and update the `/etc/resolve.conf` with the 
ip and nodes for your setup.

### ssh from host to the master node,
Add your ssh key to authorized files on master node to login from the host machine.

On master node, create authorized_keys file on the master node and add your public key
```bash
# mkdir ~/.ssh/
# vi ~/.ssh/authorized_keys
… add your public key …
# chmod 700 ~/.ssh
# chmod 600 ~/.ssu_authorized_keys
```

### Control Cluster from Laptop

This requires ssh access to the master node. 
Run the following command to copy the config file to the local kube folder

```bash
# scp root@master:/etc/kubernetes/admin.conf ~/.kube/config
```

# Troubleshooting
## Networking
Sometimes there are multiple interfaces on the worker node and joining can cause issues if it’s listening on the wrong interface

**Solutions**

1. Change the default route so it gets setup correctly when you join
[Stackoverflow Reference](https://superuser.com/questions/1129799/centos-7-default-route-for-static-ip-on-two-interface)
2. Another solution is to update the **kubeadm-flags.env** file and append the **node-ip** address of the worker node

Open up the environment file:
```bash
# vi /var/lib/kubelet/kubeadm-flags.env
```

Append the **node-ip** at the end
```
KUBELET_KUBEADM_ARGS="--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --node-ip=NODE_IP"
```

Restart the kubelet service on the worker node and verify by following the system log

```
# systemctl restart kubelet && journalctl -f -u kubelet
```

Also verify on master node that the worker node is ready

```
# kubectl get nodes
# kubectl get pods --all-namespaces
```

## kubeadm join token expired

Sample error message when joining with `--v=5`

> [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID

If you are trying to join the worker node with an expired token, you can run the following command on the master node to get a new token.

```bash
kubeadm token create
```
