## Docker

```sh
# remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Set up the repository
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo service docker start

# Verify
sudo docker run hello-world

#check the version
sudo docker -v
    
```



## K8s

```sh
# Install Kubernetes Servers
sudo apt update
sudo apt -y full-upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f

# sudo apt update
sudo apt-get update
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubelet, kubeadm and kubectl
sudo apt-get update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Confirm installation by checking the version of kubectl
kubectl version --client && kubeadm version

# Disable Swap
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a

# Enable kernel modules and configure sysctl
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system

# Installing Docker runtime
# Add repo and Install packages
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli

# Create required directories
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Start and enable Services
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker

# set docker Cgroup Driver as systemd
sudo sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# start the k8s
systemctl enable kubelet && systemctl start kubelet

# only do the following in the master node
# make sure that the br_netfilter module is loaded
lsmod | grep br_netfilter
sudo vim /etc/hosts

# add all the node and its address
129.69.209.196  master
129.69.209.194  node01
129.69.209.195  node02

```

init the master node  

```sh
touch ./kubeadm-config.yaml

# avoid error: [ERROR CRI]: container runtime is not running
rm /etc/containerd/config.toml
systemctl restart containerd

sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# config kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# config network
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

```

init the worker node  

```sh

# add worker node into cluster
# after kubeamd init in the master node, following command will display
# or you can get this command by using [kubeadm token create --print-join-command]
kubeadm join 129.69.209.196:6443 --token 96w55o.k09hn1805r9kwo1d --discovery-token-ca-cert-hash sha256:8095efe3fefa61af99de16e6d6ed421f453c1f9ad46527fbd854caf182569479

``` 

change worker node label  

```sh

# run in the master node
kubectl label node k8smavm01 node-role.kubernetes.io/worker=worker
kubectl label node k8smavm02 node-role.kubernetes.io/worker=worker

```

## k8s dashboard deployment

Get the yaml file:

```sh

wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml

```

Edit the yaml file to make it exposed in port 30000

```sh
vim recommended.yaml

# modify Service: type and nodePort
```

```yaml
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort #new
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000 #news
  selector:
    k8s-app: kubernetes-dashboard
---
```

```sh

kubectl apply -f recommand.yaml

kubectl get pods -ns kubernetes-dashboard -o wide
```

|                          NAME                    |   READY  |    STATUS  |  RESTARTS  |   AGE  |          IP         |    NODE    |   NOMINATED NODE |
| ------------------------------------------------ | -------- | ---------- | ---------- | ------ | ------------------- | ---------- | -----------------|
| dashboard-metrics-scraper-7bfdf779ff-72hqd       | 1/1      | Running    |      0     |   115s |   192.168.182.194   |  <none>    |     <none>       |
| kubernetes-dashboard-6cdd697d84-dzcdz            | 1/1      | Running    |      0     |   115s |   192.168.182.194   |  <none>    |     <none>       |

You can use the dashboard in the browser(Please make sure you connect to the vpn of informatik)  
if remind the connect is unsafe, please choose to ignore and continue
https://129.69.209.195:30000/#/login  

create admin-user and binding role
admin-user-sa.yaml:  
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
admin-user-rb.yaml:  
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```sh
kubectl apply -f admin-user-sa.yaml
kubectl apply -f admin-user-rb.yaml
```
Getting the token:
```sh
kubectl -n kubernetes-dashboard create token admin-user
```

copy this token to the browser  