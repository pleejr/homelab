# Homelab

Kubernetes cluster running on 10x Raspberry Pi 5 nodes, used as a self-hosted cloud replacement and platform for AI agent deployment.

## Hardware

| Component | Quantity | Details |
|-----------|----------|---------|
| Raspberry Pi 5 | 10 | 16GB RAM |
| PCIe M.2 Adapter + PoE HAT | 10 | Powers Pi via PoE, exposes NVMe slot |
| INLAND M.2 2242 NVMe SSD | 10 | 1TB, PCIe Gen 3x4 |
| NETGEAR GS308EPP | 2 | 8-port PoE+ Gigabit, 123W PoE budget |
| CAT6 Patch Cable (0.5ft) | 10 | |
| FIDECO USB-to-M.2 Docking Station | 1 | Used to flash NVMe SSDs |

## Cluster Layout

| Role | Nodes | IPs |
|------|-------|-----|
| Control plane | node01, node02, node03 | 192.168.2.30–32 |
| Workers | node04–node10 | 192.168.2.33–39 |
| MetalLB pool | — | 192.168.2.50–59 |
| Control plane VIP | — | 192.168.2.50 |
| Traefik ingress | — | 192.168.2.55 |

## Stack

| Layer | Technology |
|-------|-----------|
| Cluster bootstrap | kubeadm |
| Container runtime | containerd |
| CNI | Calico |
| Storage | Longhorn |
| Load balancer | MetalLB (L2) |
| Ingress | Traefik |
| TLS | cert-manager + Cloudflare DNS01 |
| GitOps | Argo CD (app-of-apps) |
| Monitoring | kube-prometheus-stack |
| Domain | pleejr.org |

## Repository Structure

```
homelab/
├── ansible/
│   └── k8s_cluster/
│       ├── deploy_k8s_cluster.yml   # main cluster bootstrap playbook
│       ├── install_argocd.yml        # argo cd install + app-of-apps bootstrap
│       ├── post_deployment_k8s_cluster.yml  # run after metallb is up
│       ├── post_deployment_longhorn.yml     # longhorn dep install (legacy, now in deploy)
│       ├── inventory/
│       │   └── hosts.yml             # node IPs and group assignments
│       ├── group_vars/
│       │   └── all_nodes.yml         # k8s version, calico version, IP config
│       └── roles/
│           ├── check_all_nodes/      # pre-flight MAC address uniqueness check
│           ├── prep_all_nodes/       # OS prep, containerd, kubeadm install
│           ├── longhorn/             # longhorn node prerequisites
│           ├── init_cluster/         # kubeadm init on primary node
│           ├── install_cni_calico/   # calico CNI deployment
│           ├── join_nodes/           # control plane + worker node join
│           ├── check_cluster/        # cluster readiness poll
│           └── post_deployment/      # switch cluster-endpoint to metallb VIP
└── argocd/
    └── apps/
        ├── appofapps.yml             # app-of-apps root application
        ├── argocd/                   # argo cd dashboard ingress + TLS
        ├── certmanager/              # cert-manager + cloudflare clusterissuer
        ├── kps/                      # kube-prometheus-stack + grafana
        ├── kubesystem/               # kube-apiserver loadbalancer service
        ├── longhorn/                 # longhorn storage + dashboard ingress
        ├── metallb/                  # metallb + IP address pool
        ├── metricsserver/            # metrics-server
        └── traefik/                  # traefik ingress + dashboard
```

## Deployment

### Prerequisites

- Ubuntu 24.04 Server arm64 flashed to all NVMe SSDs via the FIDECO docking station
- DHCP reservations set on the switches (192.168.2.30–39, keyed to each node's MAC address)
- Ansible installed on your local machine
- SSH key deployed to all nodes

### 1. Bootstrap the cluster

```bash
cd ansible/k8s_cluster
ansible-playbook deploy_k8s_cluster.yml
```

This will:
1. Add node entries to your local `/etc/hosts`
2. Verify unique MAC addresses on all nodes
3. Prepare all nodes (swap off, kernel modules, containerd, kubeadm, longhorn prereqs)
4. Initialize the control plane on node01
5. Install Calico CNI
6. Join remaining control plane and worker nodes
7. Poll until the cluster is ready

### 2. Install Argo CD

```bash
ansible-playbook install_argocd.yml
```

This installs Argo CD and applies the app-of-apps manifest, which triggers GitOps sync of all platform services.

### 3. Wait for platform services to sync

Argo CD will sync in wave order:

| Wave | Apps |
|------|------|
| 0 | prometheus-operator-crds |
| 1 | metallb, traefik, cert-manager, longhorn, kps, metrics-server |
| 2 | resource configs (IP pools, ingress routes, cluster issuers) |

### 4. Switch cluster-endpoint to load balancer

Once MetalLB is running and the kube-apiserver LoadBalancer service is assigned `192.168.2.50`:

```bash
ansible-playbook post_deployment_k8s_cluster.yml
```

This updates `/etc/hosts` on all nodes to resolve `cluster-endpoint` to the MetalLB VIP instead of the primary node's IP.

## Accessing Services

All services are exposed via Traefik at `192.168.2.55` with TLS certificates issued by Let's Encrypt through Cloudflare DNS01 challenge.

| Service | URL |
|---------|-----|
| Argo CD | https://argocd.local.pleejr.org |
| Traefik | https://traefik.local.pleejr.org |
| Longhorn | https://longhorn.local.pleejr.org |
| Grafana | https://grafana.local.pleejr.org |

## Configuration

Key variables are in `ansible/k8s_cluster/group_vars/all_nodes.yml`:

```yaml
kubernetes_version: "v1.32"
calico_version: "v3.29.2"
pod_network_cidr: "10.244.0.0/16"
control_plane_endpoint_name: "cluster-endpoint"
control_plane_endpoint_lb_ip: "192.168.2.50"
```

Helm chart versions are pinned in each app manifest under `argocd/apps/`.
