# Homelab

A three-node Kubernetes homelab for learning infrastructure automation, networking, monitoring, and reliable application deployment. Host configuration is managed with Ansible; Kubernetes manifests are kept alongside it in Git.

## Current state

- Three Ubuntu Server nodes joined and reported Ready.
- SSH key authentication and Tailscale remote access configured.
- Ansible host preparation and container runtime configuration completed.
- GitHub Actions checks Ansible changes.
- Flannel pod networking deployed.
- Two Nginx replicas deployed and reachable from the home network.
- Pod replacement after deletion tested successfully.
- HTTP readiness probe included in the Nginx manifest.
- Helm v4.3.0 installed and cluster access verified.
- Monitoring release deployed with kube-prometheus-stack 91.4.1; Grafana login verified through port forwarding and an SSH tunnel.

Status reflects repository configuration and checks performed during setup, not continuous health monitoring.

## Hardware and node roles

Each Lenovo ThinkCentre mini PC has four logical CPUs, approximately 8 GB RAM, and a 256 GB SSD. Nodes connect through a Gigabit Ethernet switch.

| Node | LAN address | Role |
| --- | --- | --- |
| node01 | 192.168.1.201 | Kubernetes control plane and Ansible control node |
| node02 | 192.168.1.202 | Kubernetes worker |
| node03 | 192.168.1.203 | Kubernetes worker |

The cluster has one control-plane node. Two application replicas do not make the control plane highly available.

Versions observed during setup:

| Component | Version |
| --- | --- |
| Ubuntu Server | 24.04.5 LTS |
| Kubernetes | v1.36.4 |
| containerd | 2.2.1 |
| Flannel image in committed manifest | v0.28.9 |
| Helm | v4.3.0 |
| kube-prometheus-stack chart | 91.4.1 |

## Network and remote access

| Setting | Value |
| --- | --- |
| Home LAN | 192.168.1.0/24 |
| Router and IPv4 DNS | 192.168.1.254 |
| Ethernet interface | eno1 |
| Kubernetes API | 192.168.1.201:6443 |
| Pod network | 10.244.0.0/16 |
| Service network | 10.96.0.0/12 |
| Demo website | http://192.168.1.201:30080 |

Stable LAN addresses are assigned using router DHCP reservations. Ubuntu remains a DHCP client; its DHCP identifier was set to the interface MAC address so the router could reserve addresses consistently.

Nodes on the same LAN communicate through the switch. Flannel uses VXLAN for pod networking and explicitly selects `eno1` with `--iface=eno1`. Tailscale provides remote administration access; cluster communication uses the LAN addresses.

SSH keys and client aliases allow connections such as `ssh node01`. Aliases are configured on the client and are not cluster-wide DNS records.

## Repository guide

| Path | Purpose |
| --- | --- |
| [ansible/inventory.ini](ansible/inventory.ini) | Hosts, connection settings, and Ansible user |
| [ansible/baseline.yml](ansible/baseline.yml) | Base operating-system maintenance and tools |
| [ansible/passwordless-sudo.yml](ansible/passwordless-sudo.yml) | Sudo configuration for administration |
| [ansible/kubernetes-prep.yml](ansible/kubernetes-prep.yml) | Disable swap, enable forwarding, and load br_netfilter |
| [ansible/containerd.yml](ansible/containerd.yml) | Configure containerd and systemd cgroups |
| [ansible/kubernetes-install.yml](ansible/kubernetes-install.yml) | Install Kubernetes tools and hold package upgrades |
| [kubernetes/kubeadm-init.yml](kubernetes/kubeadm-init.yml) | Control-plane initialization settings |
| [kubernetes/networking/kube-flannel.yml](kubernetes/networking/kube-flannel.yml) | Flannel networking resources |
| [kubernetes/demo-web.yml](kubernetes/demo-web.yml) | Nginx Deployment and NodePort Service |
| [kubernetes/monitoring/values.yml](kubernetes/monitoring/values.yml) | Monitoring retention and resource settings |
| [.github/workflows/ci.yml](.github/workflows/ci.yml) | Ansible validation workflow |

The inventory uses a local connection for node01, so the current playbooks are intended to run from node01. The other two nodes are managed over SSH.

## Completed setup milestones

1. **Operating systems and access:** installed Ubuntu Server, enabled SSH, configured SSH keys and stable addresses, and tested Tailscale access.
2. **Host automation:** configured Ansible inventory, baseline maintenance, and passwordless sudo.
3. **Continuous integration:** added GitHub Actions checks on pushes, pull requests, and manual runs.
4. **Kubernetes preparation:** disabled swap persistently, enabled IPv4 forwarding, loaded `br_netfilter` immediately and at boot, and configured containerd with systemd cgroups.
5. **Cluster bootstrap:** initialized node01 with kubeadm, installed Flannel, and joined node02 and node03 as workers.
6. **First workload:** deployed two Nginx replicas, observed one on each worker, accessed the website through node01's NodePort, and verified replacement of a deleted pod.

The Nginx manifest also includes an HTTP readiness probe for `/` on port 80. Readiness controls whether a pod receives Service traffic; it does not restart the container. Two replicas alone do not guarantee placement on separate workers.

## Routine commands

Run these from node01.

Check Ansible connectivity:

```bash
cd ~/homelab/ansible
ansible homelab -i inventory.ini -m ansible.builtin.ping
```

Inspect cluster health and workload placement:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -l app=demo-web -o wide
```

Validate and apply the demo configuration:

```bash
cd ~/homelab
kubectl apply --dry-run=server -f kubernetes/demo-web.yml
kubectl apply -f kubernetes/demo-web.yml
kubectl rollout status deployment/demo-web --timeout=180s
```

Check Flannel:

```bash
kubectl rollout status daemonset/kube-flannel-ds -n kube-flannel --timeout=180s
```

View application logs:

```bash
kubectl logs -l app=demo-web --tail=50 --prefix
```

## Monitoring

Helm release `monitoring` is installed in namespace `monitoring`. Helm reported revision 1 deployed, and Grafana browser access was tested successfully. Metric coverage and alert delivery still need verification.

The committed values configure Prometheus for seven-day retention and a default 30-second scrape interval, with resource requests and limits for Prometheus, Grafana, and Alertmanager.

Persistent storage has not been configured. Pod replacement can lose metric history and Grafana changes stored only in its local database. Chart-provisioned dashboards can be recreated.

To install or update from node01 with Helm 4:

```bash
cd ~/homelab
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --version 91.4.1 --namespace monitoring --create-namespace \
  --values kubernetes/monitoring/values.yml \
  --wait --timeout 10m --rollback-on-failure
```

Check the release and pods:

```bash
helm list -n monitoring
kubectl get pods -n monitoring -o wide
```

For temporary Grafana access, keep this command running on node01:

```bash
kubectl port-forward -n monitoring service/monitoring-grafana 3000:80
```

In a terminal on the desktop, keep an SSH tunnel open:

```bash
ssh -N -L 3000:127.0.0.1:3000 node01
```

Open http://localhost:3000 on the desktop. Use the Grafana account; do not store its password in Git. These forwarding sessions are temporary and must be restarted after they end.

An HDMI dashboard on node01 is planned. Grafana can remain in Kubernetes, while a local graphical session and browser display it on the attached monitor.

## Git and CI workflow

Edit configuration, validate it, commit, and push. GitHub Actions runs a syntax check for `ansible/baseline.yml` and `ansible-lint ansible/` on a GitHub-hosted runner.

CI currently validates Ansible only. It does not deploy to the cluster or validate the Kubernetes manifests. Deployment remains manual.

After a README update made directly on GitHub, bring the change into the node01 checkout before further work:

```bash
cd ~/homelab
git pull --ff-only
```

Keep SSH private keys, kubeconfig/admin.conf, passwords, Tailscale authentication keys, and kubeadm join tokens out of Git. The local Kubernetes administrator configuration stays in `~/.kube/config`.

## Next milestones

- [x] Install and verify Helm.
- [x] Deploy Prometheus and Grafana with settings sized for the available hardware.
- [ ] Configure persistent storage and backups.
- [ ] Back up and migrate Grafana, Prometheus, and Alertmanager local-path data to node01, then pin those monitoring workloads to node01 so either worker can be powered down without losing the dashboard.
- [ ] Later convert the cluster to three control-plane nodes and replace local-path storage with replicated storage such as Longhorn for true node-failure recovery.
- [ ] Add alerts and test failure scenarios.
- [ ] Extend CI to validate Kubernetes configuration.
- [ ] Add automated deployment from Git.
- [ ] Build a dashboard suitable for a small homelab display.
