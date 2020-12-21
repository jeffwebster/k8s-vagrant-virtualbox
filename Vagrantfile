# My default gateway is 192.168.1.254

IMAGE = "bento/ubuntu-18.04"
MASTER_MEM = 4096
MASTER_CPUS = 2
NODE_MEM = 4096
NODE_CPUS = 2
NODE_COUNT = 3
K8S_VER = "1.19.5-00"
VM_SUFFIX = "v1_19"


Vagrant.configure("2") do |config|
  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.provider :virtualbox do |vb|
      vb.name = "master-#{VM_SUFFIX}"
      vb.memory = MASTER_MEM
      vb.cpus = MASTER_CPUS
    end
    master.vm.box = IMAGE
    master.vm.hostname = "master"
    master.vm.network :public_network, ip: "192.168.1.200"
    master.vm.network "forwarded_port", guest: 22, host: 2222
    master.vm.network "forwarded_port", guest: 6443, host: 6443  
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "node#{i}-#{VM_SUFFIX}"
        vb.memory = NODE_MEM
        vb.cpus = NODE_CPUS
      end
      node.vm.box = IMAGE
      node.vm.hostname = "node#{i}"
      node.vm.network :public_network, ip: "192.168.1.#{i + 200}"
      node.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.1.#{i + 200}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
cat /vagrant/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
# bridged traffic to iptables is enabled for kube-router.
cat >> /etc/ufw/sysctl.conf <<EOF
net/bridge/bridge-nf-call-ip6tables = 1
net/bridge/bridge-nf-call-iptables = 1
net/bridge/bridge-nf-call-arptables = 1
EOF

set -e
IFNAME=$1
ADDRESS="$(ip -4 addr show $IFNAME | grep "inet" | head -1 |awk '{print $2}' | cut -d/ -f1)"
sed -e "s/^.*${HOSTNAME}.*/${ADDRESS} ${HOSTNAME} ${HOSTNAME}.local/" -i /etc/hosts

# remove ubuntu-bionic entry
sed -e '/^.*ubuntu-bionic.*/d' -i /etc/hosts

# Patch OS
apt-get update && apt-get upgrade -y

# Create local host entries
echo "192.168.1.200 master" >> /etc/hosts
echo "192.168.1.201 node1" >> /etc/hosts
echo "192.168.1.202 node2" >> /etc/hosts
echo "192.168.1.203 node3" >> /etc/hosts

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
#apt-get -qq install -y kubelet=1.19.5-00 kubeadm=1.19.5-00 kubectl=1.19.5-00
apt-get -qq install -y kubelet=#{K8S_VER} kubeadm=#{K8S_VER} kubectl=#{K8S_VER}
apt-mark hold kubelet kubectl kubeadm

# Set external DNS
sed -i -e 's/#DNS=/DNS=8.8.8.8/' /etc/systemd/resolved.conf
service systemd-resolved restart
SCRIPT

$provision_master_node = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
KEY_FILE=/vagrant/id_rsa.pub
rm -rf $OUTPUT_FILE
rm -rf $KEY_FILE

# Create key
ssh-keygen -q -t rsa -b 4096 -N '' -f /home/vagrant/.ssh/id_rsa
cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
cat /home/vagrant/.ssh/id_rsa.pub > ${KEY_FILE}

# Start cluster
sudo kubeadm init --apiserver-advertise-address=192.168.1.200 --pod-network-cidr=10.244.0.0/16 | grep -Ei "kubeadm join|discovery-token-ca-cert-hash" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl for vagrant and root users
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown -R root:root /root/.kube

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.1.200"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Use our flannel config file so that routing will work properly
kubectl create -f /vagrant/kube-flannel.yml

# Set alias on master for vagrant and root users
echo "alias k=/usr/bin/kubectl" >> $HOME/.bash_profile
sudo echo "alias k=/usr/bin/kubectl" >> /root/.bash_profile

# Install the etcd client
sudo apt install etcd-client

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL