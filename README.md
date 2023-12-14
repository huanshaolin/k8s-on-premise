## 1.Setup k8s with cilium
### 1.1. Prepare
    1 vps server 1cpu, 2gb RAM, Ubuntu Server 20.04 LTS
    2 vps server 2cpu, 4gb RAM, Ubuntu Server 20.04 LTS
    helm command line version >= 3.8

### 1.2. Setup environment k8s
Every node, we need to run command below:
```sh
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
 "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
 "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
 sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

cat <<EOF | sudo tee -a /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
EOF

sudo sed -i 's/^disabled_plugins \=/\#disabled_plugins \=/g' /etc/containerd/config.toml

sudo mkdir -p /opt/cni/bin/
sudo wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz

systemctl enable containerd
systemctl restart containerd

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

sudo apt-get install -y apt-transport-https ca-certificates curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add

echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >> ~/kubernetes.list
sudo mv ~/kubernetes.list /etc/apt/sources.list.d

sudo apt-get update

export VERSION = "1.27.0-00"
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl

sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
sudo swapoff -a

```

### 1.3. Init k8s cluster
On master node, run command below:
```sh
export MASTER_NODE_IP = "192.168.1.210"
export K8S_POD_NETWORK_CIDR = "10.244.0.0/16"

sudo systemctl enable kubelet

kubeadm init \
 --apiserver-advertise-address=$MASTER_NODE_IP \
 --pod-network-cidr=$K8S_POD_NETWORK_CIDR \
 --ignore-preflight-errors=NumCPU \
 --skip-phases=addon/kube-proxy \
 --control-plane-endpoint $MASTER_NODE_IP 

sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "Environment=\"KUBELET_EXTRA_ARGS=--node-ip=$MASTER_NODE_IP\"" | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

```
### 1.4. Setup cilium network tools
On master node run command:
```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

sudo cilium install --version 1.14.0

```

### 1.5. Join worker node
When run command kubeadmin init in step 1.3, it will output a command join worker node, let's run that command on worker node

## 2. Provision a Load Balancer with MetaLB
### 2.1. Install MetaLB with kubectl
```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

### 2.2. Create address pool
```sh
kubectl apply -f metalb/ipaddresspool.yaml
```
### 2.3. Setup ingress nginx
```sh
kubectl apply -f ingress-nginx.yaml
```

## 3. Setup observation system
### 3.1. Install metric server
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.4/components.yaml
```

### 3.2. Install prometheus
Create namespace
```sh
kubectl create namespace observation
```
install kube promethus
```sh
helm install kube-promethus -n observation oci://registry-1.docker.io/bitnamicharts/kube-prometheus --version 8.22.8 -f observation/kube-prometheus.yaml
```

### 3.3. Install grafana
```sh
helm install grafana -n observation oci://registry-1.docker.io/bitnamicharts/grafana --version 9.6.5 -f observation/grafana.yaml
```

### 4. Bitnami chart repo
```sh
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm search repo bitnami
$ helm install my-release bitnami/<chart>
```