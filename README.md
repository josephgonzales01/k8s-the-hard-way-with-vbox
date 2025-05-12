# Kubernetes Multi-Node Cluster Setup Guide

This guide walks through setting up a Kubernetes cluster on VirtualBox VMs hosted on Windows 11, inspired by Kelsey Hightower's "Kubernetes The Hard Way" but adapted for a local environment. The setup includes 5 VMs: 2 control plane nodes, 2 worker nodes, and 1 load balancer.

## Overview

This setup creates a production-like Kubernetes environment for learning purposes with:
- High availability control plane (2 nodes)
- Dedicated worker nodes (2 nodes)
- Load balancer for the control plane API server
- Private network for secure communication

## Prerequisites

- Windows 11 machine with at least 16GB RAM and 4+ CPU cores
- [VirtualBox latest version] (https://www.virtualbox.org/wiki/Downloads)
- [Ubuntu Server 22.04 LTS ISO] (https://ubuntu.com/download/alternative-downloads)
- At least 100GB free disk space

## Architecture

```
                             ┌─────────────┐
                             │    Host     │
                             │ Windows 11  │
                             └──────┬──────┘
                                    │
                        ┌───────────┴───────────┐
                        │    VirtualBox         │
                        │    Host Network       │
                        │  (192.168.56.0/24)    │
                        └───────────┬───────────┘
                                    │
          ┌───────────┬─────────────┼─────────────┬───────────┐
          │           │             │             │           │
┌─────────▼────────┐  │  ┌──────────▼──────────┐  │  ┌────────▼─────────┐
│    k8s-worker1   │  │  │       k8s-lb        │  │  │    k8s-worker2   │
│  192.168.56.21   │  │  │     192.168.56.10   │  │  │  192.168.56.22   │
│    (Worker)      │  │  │    (Load Balancer)  │  │  │    (Worker)      │
└─────────┬────────┘  │  └──────────┬──────────┘  │  └────────┬─────────┘
          │           │             │             │           │
          │           │             │             │           │
          │           │  ┌──────────▼──────────┐  │           │
          │           │  │                     │  │           │
          │           │  │      API Server     │  │           │
          │           │  │      Endpoint       │  │           │
          │           │  │   192.168.56.10:6443│  │           │
          │           │  │                     │  │           │
          │           │  └──────────┬──────────┘  │           │
          │           │             │             │           │
┌─────────▼────────┐  │  ┌──────────▼──────────┐  │  ┌────────▼─────────┐
│   k8s-control1   │◄─┴──┤    k8s-control2     │◄─┴──│    etcd cluster   │
│  192.168.56.11   │◄────►   192.168.56.12     │◄────►  (embedded in     │
│  (Control Plane) │     │   (Control Plane)   │     │  control planes)  │
└─────────┬────────┘     └──────────┬──────────┘     └──────────────────┘
```

## Step 1: Set Up the VirtualBox Network

### Steps:
1. Open VirtualBox and go to File > Host Network Manager
2. Create a new host-only network (vboxnet0)
   - IP address: 192.168.56.1
   - Subnet mask: 255.255.255.0
   - Enable DHCP server

### Why:
- Creates an isolated network for secure cluster communication
- Allows VMs to communicate with each other and the host
- Isolates cluster traffic from your regular network

## Step 2: Create VM Templates

### Steps:
1. Create a base VM with:
   - OS: Ubuntu Server 22.04 LTS
   - RAM: 2048MB
   - Disk: 20GB VDI, dynamically allocated
   - Network: Adapter 1 as NAT, Adapter 2 as Host-only (vboxnet0)

### Why:
- Using a template saves time when creating multiple similar VMs
- NAT adapter provides internet access for package installation
- Host-only adapter creates the private Kubernetes network
- Dynamic allocation conserves disk space while allowing growth

## Step 3: Install Ubuntu on Base VM

### Steps:
1. Install Ubuntu with minimal settings
2. Update packages and install basic tools:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   sudo apt install -y curl openssh-server net-tools
   ```
3. Configure static IP on second network interface (edit /etc/netplan/00-installer-config.yaml):
   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: true
       enp0s8:
         dhcp4: false
         addresses: [192.168.56.10/24]
   ```
4. Apply network configuration and shut down:
   ```bash
   sudo netplan apply
   sudo shutdown -h now
   ```

### Why:
- Minimal installation reduces overhead
- Static IPs ensure consistent networking for Kubernetes components
- Updated packages ensure security and compatibility
- Base tools (curl, ssh) are needed for further configuration

## Step 4: Clone VMs

### Steps:
Clone the base VM 5 times with the following configurations:

1. **Load Balancer**:
   - Name: k8s-lb
   - IP: 192.168.56.10
   - Resources: 1CPU, 2GB RAM
   - In VirtualBox:
     ```
     Right-click base VM > Clone > Name: k8s-lb > 
     MAC Address Policy: Generate new MAC addresses for all network adapters > 
     Clone type: Full clone > Clone
     ```
   - After starting, update hostname:
     ```bash
     sudo hostnamectl set-hostname k8s-lb
     ```
   - Update static IP in netplan config:
     ```bash
     sudo nano /etc/netplan/00-installer-config.yaml
     # Change IP to 192.168.56.10/24
     sudo netplan apply
     ```

2. **Control Plane 1**:
   - Name: k8s-control1
   - IP: 192.168.56.11
   - Resources: 2CPU, 2GB RAM
   - In VirtualBox: Similar clone process, but set name to k8s-control1
   - After starting:
     ```bash
     sudo hostnamectl set-hostname k8s-control1
     # Update netplan with IP 192.168.56.11/24
     sudo netplan apply
     ```

3. **Control Plane 2**:
   - Name: k8s-control2
   - IP: 192.168.56.12
   - Resources: 2CPU, 2GB RAM
   - In VirtualBox: Similar clone process, but set name to k8s-control2
   - After starting:
     ```bash
     sudo hostnamectl set-hostname k8s-control2
     # Update netplan with IP 192.168.56.12/24
     sudo netplan apply
     ```

4. **Worker 1**:
   - Name: k8s-worker1
   - IP: 192.168.56.21
   - Resources: 2CPU, 2GB RAM
   - In VirtualBox: Similar clone process, but set name to k8s-worker1
   - After starting:
     ```bash
     sudo hostnamectl set-hostname k8s-worker1
     # Update netplan with IP 192.168.56.21/24
     sudo netplan apply
     ```

5. **Worker 2**:
   - Name: k8s-worker2
   - IP: 192.168.56.22
   - Resources: 2CPU, 2GB RAM
   - In VirtualBox: Similar clone process, but set name to k8s-worker2
   - After starting:
     ```bash
     sudo hostnamectl set-hostname k8s-worker2
     # Update netplan with IP 192.168.56.22/24
     sudo netplan apply
     ```

### Why:
- Different hostnames and IPs allow easy identification of each node
- Control planes need more resources for managing cluster state
- Workers need resources for running workloads
- Load balancer needs fewer resources as it only proxies traffic

## Step 5: Set Up the Load Balancer (HAProxy)

### Steps:
1. Install HAProxy on k8s-lb:
   ```bash
   sudo apt install -y haproxy
   ```

2. Configure HAProxy to balance traffic between control plane nodes by editing `/etc/haproxy/haproxy.cfg`:
   ```
   global
       log /dev/log local0
       log /dev/log local1 notice
       chroot /var/lib/haproxy
       stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
       stats timeout 30s
       user haproxy
       group haproxy
       daemon

   defaults
       log global
       mode tcp
       option tcplog
       option dontlognull
       timeout connect 5000
       timeout client 50000
       timeout server 50000

   frontend kubernetes-frontend
       bind 192.168.56.10:6443
       mode tcp
       default_backend kubernetes-backend

   backend kubernetes-backend
       mode tcp
       balance roundrobin
       option tcp-check
       server k8s-control1 192.168.56.11:6443 check fall 3 rise 2
       server k8s-control2 192.168.56.12:6443 check fall 3 rise 2
   ```

3. Enable and restart HAProxy service:
   ```bash
   sudo systemctl enable haproxy
   sudo systemctl restart haproxy
   ```

### Why:
- HAProxy provides high availability for the Kubernetes API server
- Distributes incoming API requests between multiple control plane nodes
- Critical for production-like environments where control plane availability is important
- Makes the control plane horizontally scalable and fault-tolerant

## Step 6: Configure All Nodes

### Steps:
1. Disable swap:
   ```bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```

2. Load kernel modules (overlay, br_netfilter):
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

3. Set network parameters:
   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward = 1
   EOF

   sudo sysctl --system
   ```

4. Install containerd as container runtime:
   ```bash
   sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   sudo apt install -y containerd.io
   ```

5. Configure containerd:
   ```bash
   sudo mkdir -p /etc/containerd
   containerd config default | sudo tee /etc/containerd/config.toml
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```

6. Update /etc/hosts with all node IPs:
   ```bash
   echo "192.168.56.10 k8s-lb" | sudo tee -a /etc/hosts
   echo "192.168.56.11 k8s-control1" | sudo tee -a /etc/hosts
   echo "192.168.56.12 k8s-control2" | sudo tee -a /etc/hosts
   echo "192.168.56.21 k8s-worker1" | sudo tee -a /etc/hosts
   echo "192.168.56.22 k8s-worker2" | sudo tee -a /etc/hosts
   ```

### Why:
- Swap must be disabled for kubelet to work properly
- Kernel modules are required for container networking and overlay filesystems
- Network parameters allow proper packet forwarding and filtering
- Containerd is the recommended container runtime for Kubernetes
- /etc/hosts ensures nodes can resolve each other by hostname

## Step 7: Install Kubernetes Components on All Nodes

### Steps:
1. Add Kubernetes repository and install Kubernetes components:
   ```bash
   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

### Why:
- Kubelet is the primary Kubernetes agent on each node
- Kubeadm simplifies cluster setup and node joining
- Kubectl provides command-line interface to the cluster
- Holding packages prevents unexpected updates that could break the cluster

## Step 8: Configure the First Control Plane Node

### Steps:
1. Create kubeadm configuration for HA setup on k8s-control1:
   ```bash
   cat <<EOF > kubeadm-config.yaml
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: ClusterConfiguration
   kubernetesVersion: v1.28.0
   controlPlaneEndpoint: "k8s-lb:6443"
   networking:
     podSubnet: "10.244.0.0/16"
     serviceSubnet: "10.96.0.0/12"
   EOF
   ```

2. Initialize the cluster:
   ```bash
   sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
   ```

3. Set up kubeconfig for the admin user:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

4. Install Calico network plugin:
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```

5. Save join commands for other nodes (from kubeadm init output)
   - The output will contain two join commands:
     - One for control plane nodes (with --control-plane flag)
     - One for worker nodes

### Why:
- Kubeadm config specifies control plane endpoint (load balancer)
- Initializing the first node creates the core cluster components
- Kubeconfig provides authentication for kubectl commands
- CNI plugin (Calico) enables pod networking
- Join commands contain secure tokens for adding other nodes

## Step 9: Join the Second Control Plane Node

### Steps:
1. Join the second control plane node using the control plane join command on k8s-control2:
   ```bash
   # Example (replace with actual values from step 8 output):
   sudo kubeadm join k8s-lb:6443 \
       --token abcdef.0123456789abcdef \
       --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef \
       --control-plane \
       --certificate-key 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
   ```

2. Set up kubeconfig:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

### Why:
- Multiple control plane nodes provide high availability
- The second node synchronizes etcd data with the first node
- Kubeconfig allows administrative access from the second node

## Step 10: Join Worker Nodes

### Steps:
1. Join both worker nodes (k8s-worker1 and k8s-worker2) using the worker join command:
   ```bash
   # Example (replace with actual values from step 8 output):
   sudo kubeadm join k8s-lb:6443 \
       --token abcdef.0123456789abcdef \
       --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
   ```

### Why:
- Worker nodes execute the actual workloads (pods)
- Separate workers from control plane is a best practice
- Multiple workers provide capacity and fault tolerance

## Step 11: Verify the Cluster

### Steps:
1. Check node status on any control plane node:
   ```bash
   kubectl get nodes -o wide
   ```
   All nodes should show as "Ready"

2. Check system pods:
   ```bash
   kubectl get pods -n kube-system
   ```
   All pods should show as "Running"

3. Test cluster functionality:
   ```bash
   # Create a test deployment
   kubectl create deployment nginx --image=nginx
   
   # Expose the deployment
   kubectl expose deployment nginx --port=80 --type=NodePort
   
   # Check the service
   kubectl get svc nginx
   ```

### Why:
- Verifies that all nodes joined successfully
- Confirms essential system pods are running
- Testing with a deployment ensures the cluster can schedule and run containers
- Ensures cluster is operational before deploying actual workloads

## Step 12: Install Additional Kubernetes Components (Optional)

### Steps:
1. Install Dashboard (optional):
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
   
   # Create admin user for Dashboard
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin-user
     namespace: kubernetes-dashboard
   ---
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
   EOF
   
   # Get token for dashboard login
   kubectl -n kubernetes-dashboard create token admin-user
   
   # Start proxy to access dashboard
   kubectl proxy
   ```
   Access the dashboard at: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

2. Install Metrics Server (optional):
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   
   # Verify metrics server installation
   kubectl get deployment metrics-server -n kube-system
   ```

### Why:
- Dashboard provides a web UI for cluster management
- Metrics Server enables resource usage metrics and autoscaling
- These components enhance cluster observability and management

## Troubleshooting

### Common Issues:

1. **Join command timeouts**:
   - Verify network connectivity between nodes:
     ```bash
     ping 192.168.56.10
     ```
   - Check that the load balancer is running:
     ```bash
     sudo systemctl status haproxy
     telnet 192.168.56.10 6443
     ```
   - Ensure ports are not blocked by firewalls:
     ```bash
     sudo ufw status
     # If active, allow kubernetes ports
     sudo ufw allow 6443/tcp
     sudo ufw allow 2379:2380/tcp
     sudo ufw allow 10250/tcp
     sudo ufw allow 10251/tcp
     sudo ufw allow 10252/tcp
     sudo ufw allow 10255/tcp
     ```

2. **Pod networking issues**:
   - Verify Calico pods are running:
     ```bash
     kubectl get pods -n kube-system | grep calico
     ```
   - Check node network configuration:
     ```bash
     ip addr show
     ip route
     ```
   - Ensure kernel modules are loaded:
     ```bash
     lsmod | grep br_netfilter
     lsmod | grep overlay
     ```
   - Check Calico logs:
     ```bash
     kubectl logs -n kube-system calico-node-xxxxx
     ```

3. **Control plane availability**:
   - Verify HAProxy configuration:
     ```bash
     sudo systemctl status haproxy
     sudo haproxy -c -f /etc/haproxy/haproxy.cfg
     ```
   - Check etcd cluster health:
     ```bash
     sudo ETCDCTL_API=3 etcdctl \
       --cacert=/etc/kubernetes/pki/etcd/ca.crt \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key \
       --endpoints=https://127.0.0.1:2379 \
       member list
       
     sudo ETCDCTL_API=3 etcdctl \
       --cacert=/etc/kubernetes/pki/etcd/ca.crt \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key \
       --endpoints=https://127.0.0.1:2379 \
       endpoint health
     ```
   - Ensure sufficient resources for control plane nodes:
     ```bash
     free -m
     df -h
     top
     ```

4. **Token expired**:
   - Create new token for joining nodes:
     ```bash
     # On control plane node
     kubeadm token create --print-join-command
     ```

## Advanced Topics

Once your cluster is running, consider exploring:

1. **Storage solutions**:
   - Local path provisioner
   - NFS provisioner
   - Longhorn

2. **Service mesh**:
   - Istio
   - Linkerd

3. **Monitoring**:
   - Prometheus
   - Grafana
   - ELK Stack

4. **CI/CD integration**:
   - ArgoCD
   - Jenkins
   - GitHub Actions

## References

- [Kubernetes Official Documentation](https://kubernetes.io/docs/home/)
- [Kelsey Hightower's Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Calico Documentation](https://docs.projectcalico.org/)
- [HAProxy Documentation](http://www.haproxy.org/#docs)
