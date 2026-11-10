# Homelab

A three-node Kubernetes homelab for learning infrastructure automation, networking, monitoring, and reliable application deployment. Host configuration is managed with Ansible; Kubernetes manifests are kept alongside it in Git.

<p align="center">
  <img src="docs/images/homelab-rack.jpeg" alt="Three ThinkCentre nodes in the homelab rack beneath the Grafana dashboard" width="600">
</p>

## Current state

- Three Ubuntu Server nodes running a kubeadm Kubernetes cluster with containerd and Flannel.
- Ansible manages host preparation, runtime configuration, and the node01 dashboard display; GitHub Actions validates Ansible changes.
- SSH key authentication and Tailscale provide remote administration.
- Portfolio publicly available at **[tolgabilgis.com](https://tolgabilgis.com)** through Cloudflare Tunnel and HTTPS.
- Portfolio and cloudflared Deployments each configured with two replicas and image digests.
- GitHub Actions builds and tests portfolio images, publishes to GHCR, and opens deployment PRs that auto-merge after required checks.
- Argo CD automatically reconciles `kubernetes/apps` from `main`, with pruning and self-healing enabled.
- Prometheus, Grafana, and Alertmanager deployed through kube-prometheus-stack, with local-path persistent storage configured.
- Google Calendar displayed in a dark fullscreen kiosk on node01's attached monitor, with commands to switch to Grafana and back.

Status reflects configuration and setup checks, not continuous health monitoring.

## Hardware and node roles

Each Lenovo ThinkCentre mini PC has four logical CPUs, approximately 8 GB RAM, and a 256 GB SSD. Nodes connect through a Gigabit Ethernet switch.

| Node | LAN address | Role |
| --- | --- | --- |
| node01 | 192.168.1.201 | Kubernetes control plane, application workloads, and Ansible control node |
| node02 | 192.168.1.202 | Kubernetes worker |
| node03 | 192.168.1.203 | Kubernetes worker |

The control-plane scheduling taint was removed so all three nodes can run application workloads. The cluster still has only one control-plane node; application replicas do not make the control plane highly available.

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
| Portfolio on LAN | http://192.168.1.201:30081 |
| Public portfolio | https://tolgabilgis.com |

Stable LAN addresses are assigned using router DHCP reservations. Ubuntu remains a DHCP client; its DHCP identifier was set to the interface MAC address so the router could reserve addresses consistently.

Nodes on the same LAN communicate through the switch. Flannel uses VXLAN for pod networking and explicitly selects `eno1` with `--iface=eno1`. Tailscale provides remote administration access; cluster communication uses the LAN addresses.

SSH keys and client aliases allow connections such as `ssh node01`. Aliases are configured on the client and are not cluster-wide DNS records.

Public requests reach Cloudflare over HTTPS, travel through the encrypted tunnel to a cloudflared Pod, then reach `portfolio-site:80` in the `portfolio` namespace. The Service forwards traffic to Nginx on container port 8080. The tunnel uses outbound connections, so this public access does not require router port forwarding.

The tunnel token is supplied through the `cloudflared-token` Kubernetes Secret, created separately from Git. Cloudflare hostname routes, DNS, and HTTPS settings are configured in the Cloudflare dashboard.

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
| [ansible/dashboard-display.yml](ansible/dashboard-display.yml) | Original Grafana kiosk provisioning on node01; see [Monitor kiosk](#monitor-kiosk-on-node01) for the locally installed Calendar switch |
| [kubernetes/storage/local-path-storage.yml](kubernetes/storage/local-path-storage.yml) | Node-local persistent-volume provisioner |
| [kubernetes/apps/portfolio-site.yml](kubernetes/apps/portfolio-site.yml) | Portfolio namespace, Deployment, and Service |
| [kubernetes/apps/cloudflared.yml](kubernetes/apps/cloudflared.yml) | Cloudflare Tunnel connectors |
| [kubernetes/argocd/portfolio-site.yml](kubernetes/argocd/portfolio-site.yml) | Argo CD Application and scoped AppProject |
| [.github/workflows/ci.yml](.github/workflows/ci.yml) | Ansible validation workflow |

The inventory uses a local connection for node01, so the current playbooks are intended to run from node01. The other two nodes are managed over SSH.

## Completed setup milestones

1. **Operating systems and access:** installed Ubuntu Server, enabled SSH, configured SSH keys and stable addresses, and tested Tailscale access.
2. **Host automation:** configured Ansible inventory, baseline maintenance, and passwordless sudo.
3. **Continuous integration:** added GitHub Actions checks on pushes, pull requests, and manual runs.
4. **Kubernetes preparation:** disabled swap persistently, enabled IPv4 forwarding, loaded `br_netfilter` immediately and at boot, and configured containerd with systemd cgroups.
5. **Cluster bootstrap:** initialized node01 with kubeadm, installed Flannel, and joined node02 and node03 as workers.
6. **First workload:** deployed two Nginx replicas, observed one on each worker, accessed the website through node01's NodePort, and verified replacement of a deleted pod.

7. **Monitoring and display:** deployed the monitoring stack, configured local-path persistence, and added a dark Google Calendar kiosk on node01 with reversible switching to Grafana.
8. **Application delivery:** containerized the portfolio, added image tests and publishing, and connected automated deployment PRs to Argo CD.
9. **Public access:** connected Cloudflare Tunnel and verified the portfolio through the custom domain with HTTPS.

The portfolio uses readiness and liveness probes, resource requests and limits, a non-root container, and a read-only root filesystem. Rolling updates allow one extra Pod with no unavailable replicas. Topology spreading uses `ScheduleAnyway` and groups Pods by revision: separate nodes are preferred, but placement remains flexible when fewer nodes are available.

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
kubectl get pods -n portfolio -o wide
kubectl get applications -n argocd
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
kubectl logs -n portfolio -l app.kubernetes.io/name=portfolio-site --tail=50 --prefix
kubectl logs -n portfolio -l app.kubernetes.io/name=cloudflared --tail=50 --prefix
```

## Monitoring

Helm release `monitoring` is installed in namespace `monitoring`. Grafana browser access and the Kubernetes dashboard have been tested. Alert delivery and coverage of failure scenarios still need verification.

The committed values configure Prometheus for seven-day retention and a default 30-second scrape interval, with resource requests and limits for Prometheus, Grafana, and Alertmanager.

The values configure local-path persistent volumes for Prometheus (20Gi), Grafana (2Gi), and Alertmanager (2Gi). These preserve data across Pod replacement on the same node, but do not replicate it to other nodes or replace backups. If the storage node is unavailable, the affected workload cannot simply recover its data on another node.

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

The [dashboard display playbook](ansible/dashboard-display.yml) configures a dedicated local user, LightDM/Openbox, Chromium kiosk mode, and a local Grafana port-forward service on node01. It opens the Kubernetes Dashboard by Chever. Grafana allows anonymous Viewer access in the committed values; administration remains authenticated. Grafana is not published through the portfolio tunnel.

## Monitor kiosk on node01

The attached 2560×1440 HDMI monitor now defaults to the full Google Calendar weekly view in Chromium kiosk mode. Google Calendar's native dark theme is enabled, the left sidebar is collapsed, and browser tabs and the address bar are hidden. The native view preserves individual event colors and includes the account's selected calendars.

Run these commands on node01 as `tolga`; they use sudo to manage the display service:

```bash
display-switch calendar   # Show Google Calendar
display-switch grafana    # Show the existing Grafana dashboard
display-switch login      # Open Google sign-in, then Calendar
display-switch status     # Show the selected mode and service state
display-switch restore    # Restore the original Grafana-only Openbox startup
display-switch schedule status # Check scheduled sleep and current monitor state
display-switch schedule off    # Disable scheduled sleep and wake the monitor
display-switch schedule on     # Enable scheduled sleep again
```

Use `display-switch grafana` for the everyday switch back to monitoring. The selected mode is saved across reboots. `monitor-kiosk.service` starts with the graphical session and restarts Chromium if it exits. `monitor-kiosk-refresh.timer` refreshes the browser daily at **4:05 a.m. America/New_York**, so the weekly view stays current.

The physical monitor sleeps every day from **12:00 a.m. to 8:00 a.m. America/New_York**, in both Calendar and Grafana modes. The timezone automatically handles daylight-saving changes. Only the HDMI output is disabled; the node and browser keep running, and the 2560×1440 virtual screen is retained. Keyboard or mouse activity does not re-enable the HDMI signal overnight.

`monitor-kiosk-power.timer` applies the current local-time policy every minute and after boot. The browser launcher also applies it when changing modes, so browser restarts and the 4:05 a.m. refresh do not wake the monitor overnight. To suspend this schedule and wake the monitor immediately, run `display-switch schedule off`; use `display-switch schedule on` to enable it again.

Calendar uses its own persistent browser profile. Sign in directly on the monitor when needed; the session and native appearance settings survive browser restarts. To change the theme, open Calendar's gear menu and choose **Appearance**. Browser notification popups are disabled for the calendar kiosk. Grafana keeps its original browser profile, and `grafana-port-forward.service` remains enabled.

### Installed configuration and recovery

| Location | Purpose |
| --- | --- |
| `/usr/local/bin/display-switch` | Switch display modes or restore the original setup |
| `/home/dashboard/.config/monitor-kiosk/launch` | Browser launcher |
| `/home/dashboard/.config/monitor-kiosk/mode` | Saved display mode |
| `/home/dashboard/.config/monitor-kiosk/monitor-power.py` | Eastern-time sleep policy and HDMI power control |
| `/home/dashboard/.config/monitor-kiosk/original-autostart` | Path to the backed-up original Openbox startup |
| `/home/dashboard/snap/chromium/common/calendar-kiosk` | Calendar browser profile; keep local because it contains the signed-in session |
| `/etc/systemd/system/monitor-kiosk.service` | Browser supervision and startup |
| `/etc/systemd/system/monitor-kiosk-refresh.{service,timer}` | Daily browser refresh |
| `/etc/systemd/system/monitor-kiosk-power.{service,timer}` | Monitor sleep schedule for both display modes |
| `/home/tolga/.local/share/monitor-kiosk/` | Local maintenance copies of the installed files and usage notes |

For full rollback, `display-switch restore` stops and disables the kiosk service, refresh timer, and sleep timer, wakes the monitor, restores the backed-up Openbox autostart, and restarts LightDM. The original Grafana startup is then used again.

The Calendar switch was installed locally on node01. The committed [dashboard display playbook](ansible/dashboard-display.yml) still provisions the original Grafana-only startup; it does not reproduce the Calendar setup. Before reapplying that playbook to return to the original setup, run `display-switch restore` so the two browser launch methods do not compete.

Validation on November 10, 2026: signed in on the physical monitor, confirmed native dark mode with multiple event colors and subscribed calendars, checked that the login and appearance survived browser relaunch, and exercised switching to Grafana and back to Calendar. The original startup backup and enabled systemd units were also checked. For the overnight schedule, verified HDMI off/on transitions while retaining the virtual resolution, checked that the timer is enabled, and tested midnight/8 a.m. boundaries and both daylight-saving transitions.

## Git and CI workflow

Edit configuration, validate it, commit, and push. GitHub Actions runs a syntax check for `ansible/baseline.yml` and `ansible-lint ansible/` on a GitHub-hosted runner.

This repository's CI validates Ansible; Kubernetes manifest validation is still a planned addition. Argo CD handles delivery for `kubernetes/apps`; host playbooks, monitoring Helm upgrades, and cluster bootstrap remain separate operations.

## Delivery pipeline

```mermaid
flowchart TB
    subgraph APP["portfolio-site repo"]
        A["Push to main"] --> B["GitHub Actions: build and test"]
    end
    B --> C["GHCR: publish container image"]
    subgraph INFRA["homelab repo"]
        D["PR: update image digest"] --> E["CI checks pass and PR merges"]
    end
    C --> D
    E --> F["Argo CD syncs manifests"]
    F --> G["Kubernetes rolls out ready Pods"]
    C -.->|"pull image"| G
```

### Website traffic

```mermaid
flowchart LR
    A["Visitor"] -->|"HTTPS"| B["Cloudflare"]
    B -->|"Tunnel"| C["cloudflared"]
    C --> D["Service :80"]
    D --> E["Portfolio Pods :8080"]
```

Source and build workflow: [TolgaBilgis/portfolio-site](https://github.com/TolgaBilgis/portfolio-site).

Argo CD reads Git independently of the local checkout. Make lasting application changes in this repository and merge them into `main`; self-healing can undo direct edits to managed resources. Pruning removes managed resources deleted from Git. To roll back a portfolio release, revert its deployment digest change through a PR and let Argo CD reconcile it.

The Argo CD Application and AppProject are committed, but the Argo CD installation, repository credentials, and tunnel Secret must be bootstrapped separately. Credentials and dashboard-side Cloudflare settings are not recreated by the application manifests.

After a README update made directly on GitHub, bring the change into the node01 checkout before further work:

```bash
cd ~/homelab
git pull --ff-only
```

Keep SSH private keys, kubeconfig/admin.conf, passwords, Tailscale authentication keys, and kubeadm join tokens out of Git. The local Kubernetes administrator configuration stays in `~/.kube/config`.

## Next milestones

- [x] Install and verify Helm.
- [x] Deploy Prometheus and Grafana with settings sized for the available hardware.
- [x] Configure local-path persistent storage for monitoring.
- [ ] Implement and test backups and restores.
- [ ] Back up and migrate Grafana, Prometheus, and Alertmanager local-path data to node01, then pin those monitoring workloads to node01 so either worker can be powered down without losing the dashboard.
- [ ] Later convert the cluster to three control-plane nodes and replace local-path storage with replicated storage such as Longhorn for true node-failure recovery.
- [ ] Add alerts and test failure scenarios.
- [ ] Extend CI to validate Kubernetes configuration.
- [x] Add automated portfolio deployment from Git with Argo CD.
- [x] Publish the portfolio through Cloudflare Tunnel and a custom HTTPS domain.
- [x] Configure a Grafana kiosk display on node01.
- [x] Add a native dark Google Calendar kiosk with preserved event colors and reversible Grafana switching.

Update this README after each verified milestone with the configuration, its purpose, and how it was tested. Keep planned work separate from completed work.
