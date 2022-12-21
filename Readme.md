# Creating k8s cluster from scratch and updating versions
**You need 2 nodes (1 for master and 1 for worker) with joint network. In this case 10.0.0.3 should be internal address for worker node and 10.0.0.2 for master**

## Setup this step for both instances
`echo "10.0.0.3  worker1.example.com worker1" > /etc/hosts`
`echo "10.0.0.2 cp1.example.com cp1" /etc/hosts`

*Use one for these for each instance*
`hostnamectl set-hostname cp1` - for master node
`hostnamectl set-hostname worker1` - for worker node

`modprobe br_netfilter`
`modprobe overlay`

`cat << EOF | tee /etc/modules-load.d/k8s-modules.conf
br_netfilter
overlay
EOF`

`cat << EOF |  tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF`

`sysctl --system`

`apt-get update ; apt-get install -y containerd`

`mkdir -p /etc/containerd`

`containerd config default | tee /etc/containerd/config.toml`

`sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml`

`systemctl restart containerd`

`swapoff -a`

`apt-get install -y apt-transport-https curl`

`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add`

`apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"`

`apt install -y kubeadm=1.22.0-00 kubelet=1.22.0-00 kubectl=1.22.0-00`
`apt-mark hold kubeadm kubectl kubelet`


## For master node
### Installation network addon and initiating node to cluster 

`curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml -O`
`curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml -O`

*Replacing default network in calico custom-resource mask to ours (in this example - 10.0.0.0/18)*
`sed -i 's|192.168.0.0/16|10.0.0.0/18|' custom-resources.yaml`

`kubeadm init --pod-network-cidr=10.0.0.0/18`
`mkdir -p $HOME/.kube`

`cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

`chown $(id -u):$(id -g) $HOME/.kube/config`
`kubectl create -f tigera-operator.yaml`
`kubectl create -f custom-resources.yaml`


# UPDATE to 1.23 version

## On master node
`apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.23.15-00 && \
apt-mark hold kubeadm`

`kubeadm upgrade apply v1.23.15`

`kubectl drain cp1 --ignore-daemonsets`
`apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.23.15-00 kubectl=1.23.15-00 && \
apt-mark hold kubelet kubectl`

`systemctl daemon-reload`
`systemctl restart kubelet`

`kubectl uncordon cp1`

# On worker node

`apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.23.15-00 && \
apt-mark hold kubeadm`

`kubeadm upgrade node`

`kubectl drain worker1 --ignore-daemonsets` - **this should be typed to master node**

`apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.23.15-00 kubectl=1.23.15-00 && \
apt-mark hold kubelet kubectl`

`systemctl daemon-reload`
`systemctl restart kubelet`

`kubectl uncordon worker1` - **this should be typed to master node**

**And so on until 1.26.0 version. After you update to 1.26.0 you'll have some troubles, which can be solved by upgrading containerd**


# Upgrading comtainerd

## For master node

`kubectl drain $(hostname) --ignore-daemonsets`
`curl -L https://github.com/containerd/containerd/releases/download/v${VERSION}/containerd-${VERSION}-linux-amd64.tar.gz | sudo tar -xvz -C /usr/`
`cp /etc/containerd/config.toml /etc/containerd/config.toml.bak-$(date +"%Y-%m-%dT%H:%M:%S")`
`containerd config default | sudo tee /etc/containerd/config.toml`
`systemctl enable containerd`
`systemctl restart containerd`
`kubectl uncordon cp1`


## For worker node

`kubectl drain $(hostname) --ignore-daemonsets` - **this should be typed to master node**
`curl -L https://github.com/containerd/containerd/releases/download/v${VERSION}/containerd-${VERSION}-linux-amd64.tar.gz | sudo tar -xvz -C /usr/`
`cp /etc/containerd/config.toml /etc/containerd/config.toml.bak-$(date +"%Y-%m-%dT%H:%M:%S")`
`containerd config default | sudo tee /etc/containerd/config.toml`
`systemctl enable containerd`
`systemctl restart containerd`
`kubectl uncordon cp1` - **this should be typed to master node**