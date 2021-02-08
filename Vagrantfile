Vagrant.configure("2") do |config|
# config.ssh.insert_key = false
  config.vm.provider :vmware_esxi do |esxi|
    esxi.esxi_hostname         = 'lab-srv-esx-2'
    esxi.esxi_password         = 'file:'
    esxi.esxi_virtual_network  = "DMZ"
    esxi.guest_memsize         = '4096'
    esxi.guest_numvcpus        = '4'
    esxi.local_allow_overwrite = 'True'
  end


  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.box = "generic/ubuntu1710"
    master.vm.hostname = "k8s-1"
    master.vm.network :private_network, ip: "192.168.40.70"
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
  end

  %w{k8s-3}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.box = "generic/ubuntu1710"
      worker.vm.hostname = name
      worker.vm.network :private_network, ip: "192.168.40.#{i + 71}"
      worker.vm.provision :shell, privileged: false, inline: <<-SHELL
mkdir ~/vagrant
touch ~/vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.40.#{i + 71}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
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
apt-get -qq install -y kubelet kubeadm kubectl
SCRIPT

$provision_master_node = <<-SHELL
mkdir ~/vagrant
touch ~/vagrant/join.sh
OUTPUT_FILE=~/vagrant/join.sh

# Start cluster
sudo kubeadm init --apiserver-advertise-address=192.168.40.70 --pod-network-cidr=10.244.0.0/16 | grep "kubeadm join" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.40.70"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure flannel
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml
kubectl create -f calico.yaml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL
require 'fileutils'
