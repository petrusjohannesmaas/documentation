###### Configure 3 vm's
4gb RAM, 4 cores, 20gb storage (master)
2gb RAM, 2 cores, 10gb storage x2 (workers)

###### Network Setup in VirtualBox

- Open VirtualBox and select each VM.
- Go to Settings > Network.
- Set Attached to to Bridged Adapter for all three VMs. This will allow them to obtain IP addresses from your local network (if your router has DHCP enabled) or you can assign static IPs with the following steps

##### (If you want to manually assign IP)

- ###### Find the gateway and DNS server IP
ip route # the ip adress listed after `via` is your gateway IP

cat /etc/resolv.conf # you'll see a line like nameserver 8.8.8.8 which is your DNS server IP

- ###### Create a network configuration file for each node

nano /etc/sysconfig/network-scripts/ifcfg-enp0s3 # or eth0 instead of enp0s3 depending on interface

TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
NAME=<interface-name>
DEVICE=<interface-name>
ONBOOT=yes
IPADDR=192.168.88.xxx  # Replace 'xxx' with your unique IP for each VM
NETMASK=255.255.255.0
GATEWAY=192.168.88.1
DNS1=192.168.88.1


###### Save and reset all VM's network settings and get all node IP's
systemctl restart NetworkManager
reboot 
ip a

###### Install vim 
dnf install vim

###### Disable the swap partition
vim /etc/fstab

###### Comment out the dev/mapper/rl-swap line

###### Disable SE linux
vim /etc/selinux/config 

then, change SELINUX=enforced to SELINUX=disabled

###### Set hostname on master node
hostnamectl set-hostname master

###### Repeat the above steps on the two worker nodes but adjust hostnames respectively
hostnamectl set-hostname worker01
hostnamectl set-hostname worker02

###### Configure firewal on master node ######
firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp

firewall-cmd --permanent --add-port=4789/udp

firewall-cmd --reload

###### Run this command to see the ports and configuration 
firewall-cmd --list-all

###### Configure firewal on worker node ######
firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp

firewall-cmd --permanent --add-port=4789/udp

firewall-cmd --reload

###### Add the overlay and br_netfilter kernel modules on all the nodes ######
vim /etc/modules-load.d/k8s.conf

overlay
br_netfilter

###### Add the following kernel parameters on all the nodes ######
vim /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1

sysctl --system


###### Configure our host file


###### Install container runtime containerd on all the nodes ######
yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/cen...

dnf install containerd.io -y

###### Configure containerd ######
mv /etc/containerd/config.toml /etc/containerd/config.toml.bkp
containerd config default ＞ /etc/containerd/config.toml

vim /etc/containerd/config.toml
SystemdCgroup = true

sudo systemctl restart containerd
sudo systemctl enable containerd

###### Configure kubernetes repo on all the nodes ######
vim  /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1....
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1....
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni

###### Install kubernetes tools on all the nodes ######
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

###### Enable kublet service on all the nodes ######
systemctl enable kubelet

###### Create user kube & kubernetes configuration ######
useradd -G wheel kube

passwd kube

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

###### Install Calico ######
kubectl apply -f https://raw.githubusercontent.com/pro...

###### Label worker nodes ######
kubectl label node worker01 node-role.kubernetes.io/worker=worker
