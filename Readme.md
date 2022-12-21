

10.0.0.3  worker1.example.com worker1
10.0.0.2 cp1.example.com cp1

# one of these:
hostnamectl set-hostname cp1
hostnamectl set-hostname worker1

modprobe br_netfilter
modprobe overlay

cat << EOF | tee /etc/modules-load.d/k8s-modules.conf
br_netfilter
overlay
EOF

cat << EOF |  tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

apt-get update ; apt-get install -y containerd

mkdir -p /etc/containerd

containerd config default | tee /etc/containerd/config.toml

sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml

systemctl restart containerd

swapoff -a

apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add

apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

apt install -y kubeadm=1.22.0-00 kubelet=1.22.0-00 kubectl=1.22.0-00
apt-mark hold kubeadm kubectl kubelet
# for master node

curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml -O
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml -O

sed -i 's|192.168.0.0/16|10.0.0.0/18|' custom-resources.yaml

kubeadm init --pod-network-cidr=10.0.0.0/18
mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

chown $(id -u):$(id -g) $HOME/.kube/config

kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml
# kubeadm init --pod-network-cidr=10.0.0.0/18 --apiserver-advertise-address=10.0.0.3


# UPDATE to 1.23

# on master node
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.23.15-00 && \
apt-mark hold kubeadm

kubeadm upgrade apply v1.23.15

kubectl drain cp1 --ignore-daemonsets
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.23.15-00 kubectl=1.23.15-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon cp1

# worker node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.23.15-00 && \
apt-mark hold kubeadm

kubeadm upgrade node

kubectl drain worker1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.23.15-00 kubectl=1.23.15-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon worker1

# UPDATE TO 1.24

# master node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.24.9-00 && \
apt-mark hold kubeadm

kubeadm upgrade apply v1.24.9

kubectl drain cp1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.24.9-00 kubectl=1.24.9-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon cp1

** to upgrade to version 1.24.9 option --network-plugin=cni should be removed from the file /var/lib/kubelet/kubeadm-flags.env

# worker node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.24.9-00 && \
apt-mark hold kubeadm

kubeadm upgrade node

kubectl drain worker1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.24.9-00 kubectl=1.24.9-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon worker1

# UPDATE to 1.25 

# master node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.25.5-00 && \
apt-mark hold kubeadm

kubeadm upgrade plan
kubeadm upgrade apply v1.25.5

kubectl drain cp1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.25.5-00 kubectl=1.25.5-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon cp1


# worket node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.25.5-00 && \
apt-mark hold kubeadm

kubeadm upgrade node

kubectl drain worker1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.25.5-00 kubectl=1.25.5-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon worker1


# UPDATE to 1.26

# master node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.26.0-00 && \
apt-mark hold kubeadm

kubeadm upgrade plan
kubeadm upgrade apply v1.26.0

kubectl drain cp1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.26.0-00 kubectl=1.26.0-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon cp1


# worket node

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.26.0-00 && \
apt-mark hold kubeadm

kubeadm upgrade node

kubectl drain worker1 --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.26.0-00 kubectl=1.26.0-00 && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon worker1



# upgrading comtainerd

wget https://github.com/containerd/containerd/releases/download/v1.6.14/containerd-1.6.14-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.6.14-linux-amd64.tar.gz
systemctl daemon-reload
systemctl enable --now containerd