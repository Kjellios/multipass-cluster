# **Kubernetes Cluster Setup with Multipass & Flannel CNI (macOS)**

This guide covers setting up a **Kubernetes cluster** with **one master (control plane) node** and **one worker node** using Multipass, Kubeadm, and Flannel CNI on **macOS**.

---

## **Prerequisites**
- macOS system with **Multipass installed** (`brew install multipass`).
- At least **8GB of RAM** and **20GB of free disk space**.
- Nodes must be able to communicate over the network.
- Basic familiarity with **Terminal and SSH**.

---

## **1. Install Multipass and Kubernetes Tools**
Install necessary tools on macOS:

```bash
brew install multipass kubectl
```

---

## **2. Create Virtual Machines (Master & Worker Nodes)**
Create two **Ubuntu 24.04** VMs using Multipass:

```bash
multipass launch --name kube-master --cpus 2 --mem 4G --disk 10G
multipass launch --name kube-worker --cpus 2 --mem 2G --disk 10G
```

Confirm they are running:

```bash
multipass list
```

Example output:

```
Name         State    IPv4
kube-master  Running  192.168.64.2
kube-worker  Running  192.168.64.3
```

---

## **3. Access Each Node via SSH**
Connect to the master node:

```bash
multipass shell kube-master
```

Connect to the worker node:

```bash
multipass shell kube-worker
```

---

## **4. Install Kubernetes Prerequisites (Master & Worker)**  
Run the following **on both nodes**:

```bash
# Update and upgrade system packages
sudo apt update && sudo apt upgrade -y

# Install required dependencies
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common

# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl settings for Kubernetes networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

Verify that the modules are loaded:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

---

## **5. Install Containerd (Container Runtime)**
Run **on both nodes**:

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Check that containerd is running:

```bash
systemctl status containerd
```

Expected output: `active (running)`

---

## **6. Install Kubernetes Components**
Run **on both nodes**:

```bash
# Add Kubernetes APT repository
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.gpg > /dev/null
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes tools
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Enable kubelet service
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

Verify installation:

```bash
kubeadm version
kubectl version --client
```

---

## **7. Initialize the Kubernetes Cluster (Master Only)**
Run **only on the master node**:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After initialization, you will see an output with a **`kubeadm join`** command similar to this:

```bash
kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<ca-cert-hash>
```

Before joining the worker node, let's retrieve the required values.

### **Find `<master-ip>`**
Run **on the master node**:

```bash
hostname -I | awk '{print $1}'
```

Expected output:

```
192.168.64.2
```

Use this value as `<master-ip>`.

### **Find `<token>`**
Run **on the master node**:

```bash
kubeadm token list
```

Example output:

```
TOKEN                     TTL       EXPIRES                USAGES
abcdef.1234567890abcdef   23h       <expiration-time>      authentication,signing
```

Use the **TOKEN** value from this list.

If no token exists, generate one:

```bash
kubeadm token create
```

### **Find `<ca-cert-hash>`**
Run **on the master node**:

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | awk '{print $2}'
```

Expected output:

```
abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

Use this value for `<ca-cert-hash>`.

---

## **8. Configure kubectl for the Master**
Run **only on the master**:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify that the cluster is active:

```bash
kubectl get nodes
```

Expected output:
```
NAME          STATUS   ROLES           AGE   VERSION
kube-master   Ready    control-plane   10m   v1.29.x
```

---

## **9. Install Flannel CNI (Master Only)**
Run **only on the master** to enable pod networking:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Verify CNI installation:

```bash
kubectl get pods -n kube-system
```

---

## **10. Join the Worker Node to the Cluster**
Run the following **on the worker node**, using the values found earlier:

```bash
sudo kubeadm join 192.168.64.2:6443 --token abcdef.1234567890abcdef \
  --discovery-token-ca-cert-hash sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

---

## **11. Verify the Worker Node Joined**
Run **on the master**:

```bash
kubectl get nodes
```

Expected output:

```
NAME          STATUS   ROLES           AGE   VERSION
kube-master   Ready    control-plane   10m   v1.29.x
kube-worker   Ready    <none>          2m    v1.29.x
```

---

## **12. Test Kubernetes Deployment**
Deploy an Nginx pod **to test the cluster**:

```bash
kubectl run nginx --image=nginx
kubectl get pods
```

Expected output:

```
NAME     READY   STATUS    RESTARTS   AGE
nginx    1/1     Running   0          10s
```

Once confirmed, delete the test pod:

```bash
kubectl delete pod nginx
```

---

# **Final Check**
Run **on the master**:

```bash
kubectl get nodes
```

Your Kubernetes cluster is now fully set up using **Multipass on macOS**!
