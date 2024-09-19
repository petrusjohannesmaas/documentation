###### Configure firewal on master node ######
firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp

firewall-cmd --permanent --add-port=4789/udp

firewall-cmd --reload

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
