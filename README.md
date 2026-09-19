# Ansible On-Premises Kubernetes Cluster Deployment

Production-grade Ansible playbook to deploy a fully functional Kubernetes cluster on bare-metal or VM infrastructure using **kubeadm**.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                 Control Plane                    │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ kube-api  │  │ etcd     │  │ scheduler    │  │
│  │ server    │  │          │  │ controller   │  │
│  └───────────┘  └──────────┘  └──────────────┘  │
│              master-node (1x)                    │
└──────────────────────┬──────────────────────────┘
                       │ Calico CNI (Pod Network)
         ┌─────────────┴─────────────┐
         │                           │
┌────────┴────────┐        ┌────────┴────────┐
│  Worker Node 1  │        │  Worker Node 2  │
│  ┌───────────┐  │        │  ┌───────────┐  │
│  │ kubelet   │  │        │  │ kubelet   │  │
│  │ containerd│  │        │  │ containerd│  │
│  │ kube-proxy│  │        │  │ kube-proxy│  │
│  └───────────┘  │        │  └───────────┘  │
└─────────────────┘        └─────────────────┘
```

## What This Playbook Does

1. **Common Setup** (all nodes): Disable swap, load kernel modules, configure sysctl for bridged traffic, configure firewall
2. **Container Runtime** (all nodes): Install and configure containerd with SystemdCgroup
3. **Kubernetes Packages** (all nodes): Install kubeadm, kubelet, kubectl from official Kubernetes repo
4. **Control Plane Init** (master only): Initialize cluster with kubeadm, configure kubectl, install Calico CNI
5. **Worker Join** (workers only): Retrieve join token from master and join the cluster

## Prerequisites

- **Minimum 3 nodes** (1 master + 2 workers) running Ubuntu 22.04 / 24.04 LTS
- Each node needs at least **2 CPU cores** and **2 GB RAM**
- SSH access from your Ansible control node to all target nodes
- Root or sudo privileges on all target nodes

## Quick Start

### 1. Update Inventory

Edit `inventory.ini` with your node IPs:

```ini
[control_plane]
master ansible_host=192.168.1.10

[workers]
worker-1 ansible_host=192.168.1.11
worker-2 ansible_host=192.168.1.12

[k8s_cluster:children]
control_plane
workers

[k8s_cluster:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### 2. (Optional) Customize Variables

Edit `group_vars/all.yml` to change Kubernetes version, pod CIDR, etc.

### 3. Run the Playbook

```bash
# Dry run first (check mode)
ansible-playbook -i inventory.ini site.yml --check

# Deploy the cluster
ansible-playbook -i inventory.ini site.yml

# Deploy specific role only (e.g., just the control plane)
ansible-playbook -i inventory.ini site.yml --tags control_plane
```

### 4. Verify

```bash
# SSH into master node and check
ssh ubuntu@192.168.1.10
kubectl get nodes
kubectl get pods -A
```

## Project Structure

```
ansible-k8s-cluster/
├── ansible.cfg                    # Ansible configuration
├── inventory.ini                  # Target node inventory
├── site.yml                       # Master playbook (entry point)
├── group_vars/
│   └── all.yml                    # Shared variables for all roles
└── roles/
    ├── common/                    # OS-level prerequisites
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── containerd/                # Container runtime installation
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/config.toml.j2
    ├── kubernetes/                # kubeadm, kubelet, kubectl
    │   └── tasks/main.yml
    ├── control_plane/             # Master node initialization
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    └── workers/                   # Worker node join
        └── tasks/main.yml
```

## Customization

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_version` | `1.30` | Kubernetes minor version |
| `k8s_package_version` | `1.30.*` | Package version pattern |
| `pod_network_cidr` | `192.168.0.0/16` | Pod network CIDR (matches Calico default) |
| `service_cidr` | `10.96.0.0/12` | Service network CIDR |
| `calico_version` | `3.28.0` | Calico CNI version |
| `container_runtime` | `containerd` | Container runtime |

## Security Hardening Included

- ✅ Swap disabled (required by kubelet)
- ✅ Firewall configured with only required ports open
- ✅ containerd configured with SystemdCgroup (production best practice)
- ✅ Kubernetes packages version-pinned to prevent unintended upgrades
- ✅ kubeconfig permissions restricted to the admin user
