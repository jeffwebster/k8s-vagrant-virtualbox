## Ubuntu k8s package versions
Run the following command to get the available package versions for kubelet, kubectl and kubeadm
```
apt-cache policy <kubelet|kubectl|kubeadm>
```

As of 12/18/2020 these are the latest packages available for the last several versions of k8s:

* 1.20.0-00
* 1.19.5-00
* 1.18.13-00
* 1.17.15-00
* 1.16.15-00
* 1.15.12-00
* 1.14.10-00