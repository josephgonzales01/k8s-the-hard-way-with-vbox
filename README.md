# Kubernetes Multi-Node Cluster Setup Guide

This guide walks through setting up a Kubernetes cluster on VirtualBox VMs hosted on Windows 11, inspired by Kelsey Hightower's [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) but adapted for a local environment VMs. The setup includes 5 VMs: 2 control plane nodes, 2 worker nodes, and 1 load balancer.

## Overview

This setup creates a production-like Kubernetes environment for learning purposes with:

- High availability control plane (2 nodes)
- Dedicated worker nodes (2 nodes)
- Load balancer for the control plane API server
- Private network for secure communication

## Prerequisites

- Windows 11 machine with at least 16GB RAM and 4+ CPU cores
- [VirtualBox latest version](https://www.virtualbox.org/wiki/Downloads)
- [Ubuntu Server 22.04 LTS ISO](https://ubuntu.com/download/alternative-downloads)
- At least 100GB free disk space

## Architecture

```
Host
Windows 11
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

### Before You Begin

- Verify VirtualBox virtualization is enabled in BIOS
- Disable Windows Hyper-V if present (can conflict with VirtualBox)

### Steps:

1. Open VirtualBox and go to File > Tools > Network Manager
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
   - OS: Ubuntu 22.04 LTS
   - RAM: 2048MB
   - Disk: 20GB VDI, dynamically allocated
   - Network: Adapter 1 as NAT, Adapter 2 as Host-only (vboxnet0)
   - For Adapter 2, set Promiscuous Mode to "Allow VMs" to enable communication between VMs on the same host-only network

### Why:

- Using a template saves time when creating multiple similar VMs
- NAT adapter provides internet access for package installation
- Host-only adapter creates the private Kubernetes network
- Dynamic allocation conserves disk space while allowing growth
- "Allow VMs" promiscuous mode enables inter-VM communication on the host-only network

## Step 3: Install Ubuntu on Base VM

### Steps:

1. Start the Base VM and attach the Ubuntu ISO (Settings → Storage → Optical Drive)
2. During installation:

   - Create a consistent user account (e.g., "k8sadmin") with a secure password
   - When prompted for Hostname or Servername use anyname (e.g k8s-template, k8s-temp-name etc) this will be overwritten later on step 4
   - Enable "Install OpenSSH server" when prompted
   - Select standard system utilities

3. After installation, update packages and install basic tools:

   ```bash
   sudo apt update
   sudo apt upgrade -y
   sudo apt install -y curl openssh-server net-tools
   ```

4. Set up SSH keys for password-less authentication:

   ```bash
   # Generate SSH key (press Enter for default location and blank passphrase for automation)
   ssh-keygen -t rsa -b 4096

   # Add your key to authorized keys
   cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

5. Configure static IP on second network interface:

   Newer Ubuntu versions (22.04+) may use different netplan configuration filenames. Follow these steps:

   1. Identify the correct network interfaces (typically enp0s3 for NAT, enp0s8 for host-only, but may vary)
      ```bash
      ip addr show
      ```
   2. Check existing netplan files

      ```bash
      ls /etc/netplan/
      ```

      Common filenames in newer versions:

      - 00-installer-config.yaml (if using server installer)
      - 50-cloud-init.yaml (if using cloud image)
      - 01-netcfg.yaml (older versions)

   3. Edit netplan file

      ```yaml
      network:
      version: 2
      ethernets:
        enp0s3: # NAT interface (typically first adapter)
          dhcp4: true
        enp0s8: # Host-only interface (typically second adapter)
          dhcp4: false
          addresses: [192.168.56.10/24]
          optional: true # Recommended to prevent boot delays
      ```

6. Apply network configuration:

   ```bash
   # 4. Apply the configuration
   sudo netplan apply

   # 5. Verify the IP assignment
   ip addr show enp0s8
   ```

7. Disable swap (for Kubernetes requirements):

   ```bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```

8. Load kernel modules (overlay, br_netfilter):

   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

9. Set network parameters:

   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward = 1
   EOF

   sudo sysctl --system
   ```

10. Install containerd as container runtime:

    ```bash
    sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt update
    sudo apt install -y containerd.io
    ```

11. Configure containerd:

    ```bash
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    ```

12. Install Kubernetes components:

    ```bash
      sudo apt-get update && \
      sudo apt-get install -y apt-transport-https ca-certificates curl && \
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg && \
      echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
      sudo apt-get update && \
      sudo apt-get install -y kubelet kubeadm kubectl && \
      sudo apt-mark hold kubelet kubeadm kubectl && \
      echo "Kubernetes components installed successfully!" && \
      kubeadm version && kubectl version --client && kubelet --version
    ```

13. Take a snapshot of your base VM before shutdown:
    ```bash
    sudo shutdown -h now
    ```
    In VirtualBox Manager, right-click on your VM and select "Snapshots" > "Take Snapshot" and name it "Base Config Complete"

### Why:

- SSH server allows remote management of VMs
- Static IPs ensure consistent networking for Kubernetes components
- Updated packages ensure security and compatibility
- Base tools (curl, ssh) are needed for further configuration
- SSH key setup allows password-less authentication between nodes
- Snapshot provides a recovery point before cloning VMs

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
   - After starting, Update static IP in netplan config: file name might differ see [common filenames](#L141)
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

6. **Set up SSH access between nodes**:
   On each VM, update the SSH known hosts to enable password-less SSH:
   ```bash
   # From each VM, SSH to each other VM once to add to known hosts
   # Example from k8s-control1:
   ssh k8sadmin@192.168.56.10 exit
   ssh k8sadmin@192.168.56.12 exit
   ssh k8sadmin@192.168.56.21 exit
   ssh k8sadmin@192.168.56.22 exit
   ```

### Why:

- Different hostnames and IPs allow easy identification of each node
- Control planes need more resources for managing cluster state
- Workers need resources for running workloads
- Load balancer needs fewer resources as it only proxies traffic
- SSH known hosts configuration prevents interactive prompts during automated operations

## Step 5: Set Up the Load Balancer (HAProxy)

### Steps:

**Execute these commands only on k8s-lb (192.168.56.10)**

1. Install HAProxy on k8s-lb:

   ```bash
   sudo apt install -y haproxy
   ```

2. Configure HAProxy to balance traffic between control plane nodes by editing `/etc/haproxy/haproxy.cfg`:

   ```bash
   # Back up the original configuration
   sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

   # Create new configuration file
   sudo bash -c 'cat > /etc/haproxy/haproxy.cfg' << 'EOF'
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
   EOF
   ```

3. Enable and restart HAProxy service:

   ```bash
   sudo systemctl enable haproxy
   sudo systemctl restart haproxy
   ```

4. Verify HAProxy is running and listening:

   ```bash
   sudo systemctl status haproxy
   netstat -tuln | grep 6443
   ```

5. Take a snapshot of the load balancer VM:
   ```
   # Shutdown the VM
   sudo shutdown -h now
   ```
   In VirtualBox Manager, right-click on k8s-lb VM and select "Snapshots" > "Take Snapshot" and name it "HAProxy Configured"

### Why:

- HAProxy provides high availability for the Kubernetes API server
- Distributes incoming API requests between multiple control plane nodes
- Critical for production-like environments where control plane availability is important
- Makes the control plane horizontally scalable and fault-tolerant
- Snapshot provides a recovery point in case of issues with the load balancer

## Step 6: Configure All Nodes

### Steps:

**Execute these commands on all 5 VMs (load balancer, both control planes, and both workers)**

You can use any of the access methods described in the "How to Execute Commands on Nodes" section:

- Direct console access through VirtualBox
- SSH from your Windows host: `ssh k8sadmin@192.168.56.XX`
- SSH from another VM: `ssh k8sadmin@192.168.56.XX`

1. Update /etc/hosts with all node IPs:

   ```bash
   echo "192.168.56.10 k8s-lb" | sudo tee -a /etc/hosts
   echo "192.168.56.11 k8s-control1" | sudo tee -a /etc/hosts
   echo "192.168.56.12 k8s-control2" | sudo tee -a /etc/hosts
   echo "192.168.56.21 k8s-worker1" | sudo tee -a /etc/hosts
   echo "192.168.56.22 k8s-worker2" | sudo tee -a /etc/hosts
   ```

2. Verify network interface configuration:

   ```bash
   ip addr show
   ```

   Ensure your interfaces match what's expected in your netplan configuration

3. Take a snapshot of each VM after configuration:
   ```bash
   sudo shutdown -h now
   ```
   In VirtualBox Manager, take snapshots of each VM named "Base Configuration Complete"

### Why:

- /etc/hosts ensures nodes can resolve each other by hostname
- Network verification confirms proper connectivity
- Most setup work was already done in the base VM (Step 3)
- Snapshots provide recovery points before cluster initialization

## Step 7: Install Kubernetes Components on All Nodes

### Skip this step as components were already installed on the base VM in Step 3.

## Step 8: Configure the First Control Plane Node

### Steps:

**Execute these commands only on k8s-control1 (192.168.56.11)**

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

   This will take a few minutes to complete. The output is very important - it contains the join commands for other nodes.

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

5. Save join commands for other nodes (from kubeadm init output):

   - The output will contain two join commands:
     - One for control plane nodes (with --control-plane flag)
     - One for worker nodes
   - Copy both commands to a text file for safekeeping
   - These commands contain security tokens that expire after 24 hours

6. Save certificate keys and tokens securely:

   ```bash
   # Create a secure file for tokens
   touch ~/k8s-tokens.txt
   chmod 600 ~/k8s-tokens.txt

   # Copy the entire kubeadm init output into this file
   nano ~/k8s-tokens.txt

   # Paste the kubeadm output here and save
   ```

7. Take a snapshot after control plane initialization:
   Shut down the VM and take a snapshot in VirtualBox named "Control Plane 1 Initialized"

### Why:

- Kubeadm config specifies control plane endpoint (load balancer)
- Initializing the first node creates the core cluster components
- Kubeconfig provides authentication for kubectl commands
- CNI plugin (Calico) enables pod networking
- Saving join commands securely is critical as they contain authentication tokens
- Taking a snapshot provides a recovery point

## Step 9: Join the Second Control Plane Node

### Steps:

**Execute these commands only on k8s-control2 (192.168.56.12)**

1. Transfer the join command from k8s-control1 to k8s-control2:

   ```bash
   # Option 1: If you've set up SSH keys properly, you can copy directly:
   ssh k8sadmin@192.168.56.11 "cat ~/k8s-tokens.txt" > ~/k8s-tokens.txt

   # Option 2: Use a secure method to transfer the join command
   # (manually copy/paste via SSH terminal is acceptable for lab environment)
   ```

2. Join the second control plane node using the control plane join command:

   ```bash
   # Example (replace with actual values from k8s-tokens.txt):
   sudo kubeadm join k8s-lb:6443 \
       --token abcdef.0123456789abcdef \
       --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef \
       --control-plane \
       --certificate-key 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
   ```

3. Set up kubeconfig:

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

4. Verify this node has joined the cluster:

   ```bash
   kubectl get nodes
   ```

5. Take a snapshot after joining:
   Shut down the VM and take a snapshot in VirtualBox named "Control Plane 2 Joined"

### Why:

- Multiple control plane nodes provide high availability
- The second node synchronizes etcd data with the first node
- Kubeconfig allows administrative access from the second node
- Secure token transfer is important for cluster security
- Taking a snapshot provides a recovery point

## Step 10: Join Worker Nodes

### Steps:

**Execute these commands on both worker nodes (k8s-worker1 - 192.168.56.21 and k8s-worker2 - 192.168.56.22)**

1. Transfer the worker join command from k8s-control1 to each worker node:

   ```bash
   # Option 1: If you've set up SSH keys properly, you can copy directly:
   ssh k8sadmin@192.168.56.11 "cat ~/k8s-tokens.txt" > ~/k8s-tokens.txt

   # Option 2: Use a secure method to transfer the join command
   # (manually copy/paste via SSH terminal is acceptable for lab environment)
   ```

2. Join worker nodes using the worker join command:

   ```bash
   # Example (replace with actual values from k8s-tokens.txt):
   sudo kubeadm join k8s-lb:6443 \
       --token abcdef.0123456789abcdef \
       --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
   ```

3. Verify nodes joined successfully by running this command on any control plane node:

   ```bash
   kubectl get nodes
   ```

   You should see both worker nodes listed with "Ready" status (may take a minute or two)

4. Take snapshots of worker nodes after joining:
   Shut down each VM and take snapshots in VirtualBox named "Worker Joined Cluster"

### Why:

- Worker nodes execute the actual workloads (pods)
- Separate workers from control plane is a best practice
- Multiple workers provide capacity and fault tolerance
- Taking snapshots provides recovery points

## Step 11: Verify the Cluster

### Steps:

**Execute these commands on any control plane node (k8s-control1 or k8s-control2)**

1. Check node status:

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

   Take note of the assigned NodePort (e.g., 32XXX)

4. Test accessing the deployed service:

   ```bash
   # Get the IP of a worker node
   kubectl get nodes -o wide

   # Access the service from any node (replace NODE_IP and NODE_PORT with actual values)
   curl http://NODE_IP:NODE_PORT
   ```

   You should see the Nginx welcome page HTML

### Why:

- Verifies that all nodes joined successfully
- Confirms essential system pods are running
- Testing with a deployment ensures the cluster can schedule and run containers
- Ensures cluster is operational before deploying actual workloads

## Step 12: Install Additional Kubernetes Components (Optional)

### Steps:

**Execute these commands on any control plane node (k8s-control1 or k8s-control2)**

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

   To access the dashboard:

   - From your Windows host, set up port forwarding to the control plane node:
     ```
     ssh -L 8001:localhost:8001 k8sadmin@192.168.56.11
     ```
   - Then access the dashboard at: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
   - Use the token from the previous step to log in

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

5. **Network interface name differences**:

   - If your network interfaces don't match what's in the guide:

     ```bash
     # Identify your interfaces
     ip addr show

     # Update your netplan configuration accordingly
     sudo nano /etc/netplan/00-installer-config.yaml
     # Replace interface names with your actual interface names
     sudo netplan apply
     ```

6. **Certificate issues with control plane join**:

   - If certificate key expired or is invalid:

     ```bash
     # On first control plane node (k8s-control1)
     sudo kubeadm init phase upload-certs --upload-certs
     # This will generate a new certificate key to use with the join command

     # Then create a new join command
     sudo kubeadm token create --print-join-command
     # Add the --control-plane and --certificate-key flags with the new key
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
