# KUBERNETES Installation (1.27) 

# Before you begin 

A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager. 

# Imp: In a Kubernetes cluster, while Ubuntu 20.04 is generally considered stable, users might encounter certain issues with container restarts when using Ubuntu 22.04. Also can face the below issue during the time of installation:

**error execution phase addon/kube-proxy: error when creating kube-proxy service account: unable to create serviceaccount: client rate limiter Wait returned an error: context deadline exceeded
To see the stack trace of this error execute with --v=5 or higher**

# Prerequisites:

i)   2 GB or more of RAM per machine (any less will leave little room for your apps).

ii)  2 CPUs or more.

iii) Full network connectivity between all machines in the cluster (public or private network is fine).

iv)  Unique hostname, MAC address, and product_uuid for every node. See here for more details.

v)   Certain ports are open on your machines. See the link for more details. 
    "https://kubernetes.io/docs/reference/networking/ports-and-protocols/"

vi)  Swap disabled. You MUST disable swap in order for the kubelet to work properly.
     For example, sudo swapoff -a will disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd. swap, depending on how it was configured on your system.

# 1. Forwarding IPv4 and letting iptables see bridged traffic

Ref: "[https://kubernetes.io/docs/setup/production-environment/container-runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)/"
~~~

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
~~~

**Verify that the br_netfilter, overlay modules are loaded by running the following commands**

~~~
lsmod | grep br_netfilter
lsmod | grep overlay
~~~

**Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:**
~~~
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
~~~

# 2. Install runtime (Containerd)

Ref: "https://docs.docker.com/engine/install/ubuntu/"
     "https://kubernetes.io/docs/setup/production-environment/container-runtimes/"

**Set up the repository**

 i)  Update the apt package index and install packages to allow apt to use a repository over HTTPS:
~~~
 sudo apt-get update
 sudo apt-get install ca-certificates curl gnupg
~~~

 ii)  Add Docker’s official GPG key:
~~~
 sudo install -m 0755 -d /etc/apt/keyrings
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
 sudo chmod a+r /etc/apt/keyrings/docker.gpg
~~~

iii) Use the following command to set up the repository:
~~~
 echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

~~~

iv) Update the apt package index:
~~~
 sudo apt-get update
~~~

# Install containerd
~~~
apt-get -y install containerd.io
~~~

# 3. Configuring the systemd cgroup driver 
  Ref: "https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd"

1. To use the systemd cgroup driver in **/etc/containerd/config.toml** with runc, set the below
   (**Note:** Remove all default the entries from file)
~~~
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
~~~

2. Restart the containerd service
~~~
systemctl restart containerd.service
~~~

# 4. Installing kubeadm, kubelet and kubectl
  Ref: "https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl"

1. Update the apt package index and install packages needed to use the Kubernetes apt repository:
~~~
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
~~~

2. Download the Google Cloud public signing key:
~~~
curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
~~~

3. Add the Kubernetes apt repository:
~~~
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
~~~

4. Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
~~~
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
~~~

# 5. Initializing your control-plane node 
  Ref: "https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node"
  
  The control-plane node is the machine where the control plane components run, including etcd (the cluster database) and the API Server (which the kubectl command line tool communicates with).

1. (Recommended but optional) If you have plans to upgrade this single control-plane kubeadm cluster to high availability you should specify the **--control-plane-endpoint** to set the shared endpoint for all control-plane nodes. Such an endpoint can be either a DNS name or an IP address of a load-balancer.

2. (Optional) Choose a Pod network add-on, and verify whether it requires any arguments to be passed to kubeadm init. Depending on which third-party provider you choose, you might need to set the **--pod-network-cidr** to a provider-specific value. See Installing a Pod network add-on.

3. (Optional) kubeadm tries to detect the container runtime by using a list of well-known endpoints. To use different container runtime or if there is more than one installed on the provisioned node, specify the **--cri-socket** argument to kubeadm. See Installing a runtime. Check the path of socket from the below link:
~~~
Runtime	Path to Unix domain socket
containerd	      unix:///var/run/containerd/containerd.sock
CRI-O	            unix:///var/run/crio/crio.sock
Docker Engine (using cri-dockerd)	unix:///var/run/cri-dockerd.sock
~~~

4. (Optional) Unless otherwise specified, kubeadm uses the network interface associated with the default gateway to set the advertise address for this particular control-plane node's API server. To use a different network interface, specify the --apiserver-advertise-address=<ip-address> argument to kubeadm init. To deploy an IPv6 Kubernetes cluster using IPv6 addressing, you must specify an IPv6 address, for example **--apiserver-advertise-address=2001:db8::101**
   
# To initialize the control-plane node run:
~~~
kubeadm init <args> ==> args is optional 
~~~
**Note:** Copy the token generated at the end

# 6. To start using the cluster, run the following as a regular user
~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~
# 7. To add the cni plugging:
  Ref: "https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation"
~~~
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
~~~
**NOTE:** if **--pod-network-cidr** use with kubeadm init, then specify the network range in the cni ds too using the doc [1] or **Ignore** if you didn't use that option 
[1] https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-changing-configuration-options

# 8. Perform the same steps till 4th on Worker Nodes and then Paste the token command which we copy in the end in step 5.
~~~
# Sample token

kubeadm join 172.31.31.251:6443 --token 0wpf56.v07txiyytm5emlmz \
	--discovery-token-ca-cert-hash sha256:7d658f139547cf0e51ab00bb099c92498b35e9a153e10cd07b7025a216c79f08
~~~

# Congratuations! Cluster Successfully Install.
