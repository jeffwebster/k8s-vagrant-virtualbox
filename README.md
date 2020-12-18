# k8s-vagrant-virtualbox
Create a local Kubernetes cluster using Vagrant and VirtualBox.
Modification of great work by [danielepolencic](https://github.com/danielepolencic) in his [gist](https://gist.github.com/danielepolencic/ef4ddb763fd9a18bf2f1eaaa2e337544)

## Prerequesites
Install the following
* [git client](https://git-scm.com/downloads)
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* [Vagrant](https://www.vagrantup.com/downloads.html)

## Setup
Clone this repo:
```
git clone https://github.com/jeffwebster/k8s-vagrant-virtualbox.git
```
Update the Vagrantfile settings as needed for your system:
```
IMAGE = "bento/ubuntu-18.04"
MASTER_MEM = 4096
MASTER_CPUS = 2
NODE_MEM = 4096
NODE_CPUS = 2
NODE_COUNT = 3
K8S_VER = "1.19.5-00"
VM_SUFFIX = "v1_19"
```
Change the IP addressing as needed. Current vagrant file is based on the default gateway being `192.168.1.254`. The master is assigned `192.168.1.200`, and the worked nodes are assigned addresses starting with `192.168.1.201`

## Starting the cluster
```
vagrant up
```
## Stopping the cluster
```
vagrant halt
```
## SSH into cluster
```
vagrant ssh master
vagrant ssh node1
vagrant ssh node12
vagrant ssh node3
```
## K8s Node Status
```
$ kubectl get nodes -o wide
```
## SSH to other Nodes in the cluster from the Master:

```bash
$ ssh node1
$ ssh node2
$ ssh node3
```

# Tear down cluster
```
vagrant destroy -f
```

## Troubleshooting

### kube-flannel
In kube-flannel.yml the DaemonSet container `kube-flannel` has the following command args:
```
command:
- /opt/bin/flanneld
args:
- --ip-masq
- --iface=eth1
- --kube-subnet-mgr
```
The `iface` value will vary by system. After running `vagrant up`, run the following:
```
vagrant ssh master
```
Then once you have a shell in master, run this:
```
kubectl get po --all-namespaces
```

If you see the pods starting with `kube-flannel-ds-amd64...` in Error or CrashloopBackoff status, then you will likely need to change the `iface` value. Do the following to determine the correct value:
```
ip -4 addr show
```
From the output of this command, select the interface name that is up and has the inet address range that matches with what you've assigned to the `vm.network` ip addresses in your Vagrantfile. You will then need to update your `kube-flannel.yml` file with that interface name, and update the `kube-flannel-ds-amd64` daemonset in the `kube-system` namespace. 

