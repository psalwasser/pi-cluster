# pi-cluster

A 3-node, high-availability k3s cluster on Raspberry Pi 5s, built to be **torn down
and rebuilt** for learning. Ansible sets up the nodes from bare Ubuntu, and Flux
runs everything in the cluster from this repo.

```
                    workstation (Omarchy, 10.0.0.108)
                    kubectl / k9s / flux / ansible   ── context "home" ──┐
                                                                         │ :6443
   ┌──────────────── home LAN 10.0.0.0/24 (Wi-Fi today) ─────────────────┼───────┐
   │                                                                     ▼       │
   │   pi1 10.0.0.104          pi2 10.0.0.105          pi3 10.0.0.239            │
   │   k3s server              k3s server              k3s server                │
   │   etcd (bootstrap)  ◄──►  etcd              ◄──►  etcd                      │
   │   Traefik / ServiceLB on :80/:443 of every node                             │
   └─────────────────────────────────────────────────────────────────────────────┘
          ▲ hourly: stage snapshots + token for read-only pull
          │
   omarchy 192.168.6.13 ── every 6h: pulls etcd snapshots → ~/Backups/pi-cluster/

   GitHub psalwasser/pi-cluster (public) ── Flux pulls manifests/ every minute
```

- **All three nodes are k3s servers** with embedded etcd (control-plane + etcd + worker,
  no taints). Any one node can fail and the cluster keeps running.
- **k3s defaults are kept:** Traefik (ingress), ServiceLB, local-path storage, metrics-server.
- **Only reachable from the home LAN.** Nothing is exposed to the internet.

## Inventory

| Node | IP | Wi-Fi MAC (`wlan0`, reserved today) | Ethernet MAC (`eth0`) |
|---|---|---|---|
| pi1 | 10.0.0.104 | 88:a2:9e:79:3e:6c | 88:a2:9e:79:3e:6b |
| pi2 | 10.0.0.105 | 88:a2:9e:7a:12:3b | 88:a2:9e:7a:12:3a |
| pi3 | 10.0.0.239 | 88:a2:9e:7a:23:06 | 88:a2:9e:7a:23:05 |

| | |
|---|---|
| Hardware | Raspberry Pi 5 (4 CPUs, 16 GB), 119 GB SD card, official M.2 HAT+ (no NVMe yet) |
| OS | Ubuntu Server 24.04.5 LTS (arm64), kernel 6.8.0-raspi |
| k3s | `v1.36.6+k3s1`, pinned in `ansible/inventory/group_vars/all.yml` |
| Flux | `v2.9.6`, pinned in `manifests/clusters/home/flux-system/gotk-components.yaml` |
| Network | Wi-Fi (`k3s_iface: wlan0`), DHCP reservations on the router, Wi-Fi power saving off |
| etcd tuning | heartbeat 500 ms, election timeout 5 s (for Wi-Fi + SD cards) |
| Firewall | ufw is **off** on the nodes (the role adds k3s rules only if it is on) |
| SSH | key-only, `pts` user, passwordless sudo for Ansible |

## Repo layout

```
ansible/
  inventory/hosts.yml           nodes + backup host
  inventory/group_vars/all.yml  interface, LAN, k3s version + install script sha256
  playbooks/
    00-preflight.yml            read-only node report
    01-bootstrap-sudo.yml       one-time: passwordless sudo (asks for the password)
    10-base.yml                 OS: updates, time sync, SSH lockdown, Wi-Fi power save, unattended-upgrades
    20-k3s.yml                  k3s servers, one at a time (pi1 bootstraps etcd)
    30-workstation.yml          merges the kubeconfig into ~/.kube/config as context "home"
    40-backup.yml               etcd snapshot backups pulled to omarchy
    50-flux.yml                 installs Flux (only if it is not already there)
    99-teardown-k3s.yml         DESTRUCTIVE: removes k3s and all cluster data
  roles/ base, k3s, backup_source, backup_target
manifests/
  clusters/home/flux-system/    Flux itself + "watch this repo"
  clusters/home/infrastructure.yaml → manifests/infrastructure/
  clusters/home/apps.yaml           → manifests/apps/   (waits for infrastructure)
docs/restore.md                 etcd restore procedure
```

All playbooks are run from `ansible/` and can be rerun safely: a second run changes nothing.

## Secrets: where they live (none are in git)

| Secret | Location | How to recreate |
|---|---|---|
| k3s server/join token | `~/.config/pi-cluster/k3s-token` (workstation, 0600); `/var/lib/rancher/k3s/server/token` on each node; omarchy `~/Backups/pi-cluster/<node>/token` | A new cluster makes a new one. `20-k3s.yml --limit pi1` fetches it. |
| Cluster admin kubeconfig | `~/.kube/home.yaml` and merged into `~/.kube/config` (0600) | `30-workstation.yml` |
| Backup pull key | omarchy `~/.ssh/pi-cluster-backup` | `40-backup.yml` makes it if missing |
| `pts` sudo password | you | Needed only for `01-bootstrap-sudo.yml` |
| Wi-Fi password | `/etc/netplan/50-cloud-init.yaml` on each node (set by Raspberry Pi Imager) | Imager |

The GitHub repo is **public**: never commit Secrets. For app secrets, see
"Phase-2 ideas" (SOPS) or create them by hand with `kubectl create secret`.

## Daily use

```sh
kubectl --context home get nodes
k9s --context home
flux --context home get all -A          # what Flux has applied, at which commit
flux --context home events              # what Flux is doing
flux --context home reconcile kustomization flux-system --with-source   # "sync now"
```

To deploy something, add manifests under `manifests/apps/<name>/`, list the folder
in `manifests/apps/kustomization.yaml`, commit, and push. Flux applies it within about a
minute. Delete it from git and Flux removes it from the cluster (`prune: true`).

Apps are reachable through Traefik on port 80/443 of **any** node IP (for example `http://10.0.0.105/`).

## Rebuild from scratch

### A. Keep the OS, rebuild only Kubernetes

```sh
cd ansible
ansible-playbook playbooks/99-teardown-k3s.yml -e confirm_teardown=yes   # wipes ALL cluster data
ansible-playbook playbooks/20-k3s.yml --limit pi1
ansible-playbook playbooks/20-k3s.yml --limit pi2
ansible-playbook playbooks/20-k3s.yml --limit pi3
ansible-playbook playbooks/30-workstation.yml
ansible-playbook playbooks/50-flux.yml          # Flux then restores every app from git
```

Data in local-path volumes is **not** restored: it was deleted. To get cluster state back
instead of a clean cluster, use [docs/restore.md](docs/restore.md).

### B. Reimage the SD cards

1. **Raspberry Pi Imager** → Ubuntu Server 24.04 LTS (64-bit). In the settings, set **a
   different hostname per card** (`pi1`, `pi2`, `pi3`), user `pts` + password, your
   Wi-Fi, and **SSH with public-key auth** using `~/.ssh/id_ed25519.pub`.
2. Boot them. The router reservations give them the same IPs.
3. Forget the old SSH host keys:
   `for ip in 10.0.0.104 10.0.0.105 10.0.0.239; do ssh-keygen -R $ip; ssh -o StrictHostKeyChecking=accept-new pts@$ip true; done`
4. Run the playbooks in order (from `ansible/`):
   ```sh
   ansible-playbook playbooks/00-preflight.yml
   ansible-playbook playbooks/01-bootstrap-sudo.yml -K           # one time, asks for the sudo password
   ansible-playbook playbooks/10-base.yml -e base_allow_reboot=true
   ansible-playbook playbooks/20-k3s.yml --limit pi1             # then pi2, then pi3
   ansible-playbook playbooks/30-workstation.yml
   ansible-playbook playbooks/40-backup.yml                      # refreshes host keys on omarchy too
   ansible-playbook playbooks/50-flux.yml
   ```

## Upgrade k3s (one node at a time)

Practice target: `v1.36.6+k3s1` → `v1.37.x`. Move up **one minor version at a time**
and read the release notes first. Take a snapshot before you start
(`ssh pts@10.0.0.104 sudo k3s etcd-snapshot save --name before-upgrade`).

1. Pick a version from `curl -s https://update.k3s.io/v1-release/channels | jq` and update
   **both** values in `ansible/inventory/group_vars/all.yml`:
   ```yaml
   k3s_version: v1.37.2+k3s1
   k3s_install_script_sha256: <curl -sfL https://raw.githubusercontent.com/k3s-io/k3s/v1.37.2+k3s1/install.sh | sha256sum>
   ```
2. For **pi1, then pi2, then pi3**:
   ```sh
   kubectl --context home drain piN --ignore-daemonsets --delete-emptydir-data
   ansible-playbook playbooks/20-k3s.yml --limit piN       # installs the new version, waits for Ready
   kubectl --context home uncordon piN
   kubectl --context home get nodes                        # piN shows the new version, all Ready
   ```
3. Commit the version change.

Pods using local-path volumes cannot move to another node. They wait until their node is back.

## Patch the OS

- **Security updates** (not kernel) install themselves daily (unattended-upgrades, no reboot).
- **Kernel and everything else**, one node at a time:
  ```sh
  kubectl --context home drain piN --ignore-daemonsets --delete-emptydir-data
  ansible-playbook playbooks/10-base.yml --limit piN -e base_allow_reboot=true
  kubectl --context home uncordon piN
  ```
  Without `-e base_allow_reboot=true` the role only warns that a reboot is pending.

## Restore etcd

See [docs/restore.md](docs/restore.md). **Practise on a throwaway VM, never on this cluster.**
Backups: k3s snapshots every 12 h (keeps 5) on each node → staged hourly → pulled to
omarchy `~/Backups/pi-cluster/` every 6 h (keeps 30 days). The omarchy timer runs only while
`pts` is logged in there (linger is off) and catches up on login. Manual snapshot:
`ssh pts@10.0.0.104 sudo k3s etcd-snapshot save --name <name>`.

## Add a 4th node

etcd wants an **odd** number of members: 4 servers tolerate the same single failure as 3.
Add a 4th as a **server** only on the way to 5. Otherwise add it as an **agent** (worker only),
which needs a small `k3s_agents` addition to the k3s role (`k3s-install.sh agent` with `server:`
and `token:`). That isn't written yet.

For either:
1. Image it as in "Reimage" (hostname `pi4`), reserve its IP, and add it to
   `ansible/inventory/hosts.yml` (under `k3s_servers` or a new `k3s_agents` group).
2. `01-bootstrap-sudo.yml -K --limit pi4`, then `10-base.yml --limit pi4 -e base_allow_reboot=true`.
3. Join it: `20-k3s.yml --limit pi4`.
4. The API certificate names come from the inventory, so the existing servers' config changes. Roll
   it out one at a time: `20-k3s.yml --limit pi1` (restarts k3s), wait for Ready, then pi2, pi3.
5. `40-backup.yml` to include it in backups.

## Move to Ethernet

The IPs stay the same, so etcd and the certificates don't notice.
For **one node at a time**:
1. In the router, move the node's reservation from its Wi-Fi MAC to its **Ethernet MAC** (table above).
2. Plug in the cable and reboot the node. eth0 takes the reserved IP; Wi-Fi gets some other
   address and stays as a backup way in.
3. When every node is cabled, set `k3s_iface: eth0` in `group_vars/all.yml` and run
   `20-k3s.yml --limit pi1`, then pi2, then pi3 (each restarts k3s and waits for Ready).
4. Optional: lower the etcd timeouts back toward the defaults (`k3s_etcd_heartbeat_ms`,
   `k3s_etcd_election_ms` in `roles/k3s/defaults/main.yml`), again one node at a time.

## Move to NVMe

Requires 2230/2242 M-key NVMe drives in the M.2 HAT+. The EEPROM boot order today is
`0xf461` (SD card, then NVMe, then USB).
- **Per node (no downtime):** drain the node, `kubectl delete node piN`, run `sudo k3s-uninstall.sh` on it,
  image Ubuntu onto the NVMe, set `BOOT_ORDER=0xf416` (`sudo rpi-eeprom-config --edit`), remove
  the SD card, then follow "Reimage" steps 3–4 for that node only.
- **Or:** treat it as a full rebuild (B) plus a restore.

## Upgrade Flux

Flux manages itself from git, so never `kubectl apply` its files by hand on a running cluster.
```sh
# install the newer flux CLI first (e.g. 2.10.x), then:
flux install --version=v2.10.0 --export > manifests/clusters/home/flux-system/gotk-components.yaml
git commit -am "flux: v2.10.0" && git push
flux --context home get kustomizations -A    # flux-system applies the new controllers
```

## Things learned during the build

- **Cloned images and cloud-init.** All three cards first came up as `pi2`. cloud-init cached the
  image's hostname and reset it at every boot. The base role disables cloud-init after first boot.
- **Wi-Fi is the weak spot.** pi1 dropped off Wi-Fi (brcmfmac firmware timeout) during an automatic upgrade
  and had to be power-cycled. The upgrade was interrupted, which left `openssh-server` half-installed.
  The base role now repairs interrupted dpkg runs and turns Wi-Fi power saving off.
- **ufw "active" vs active.** `systemctl is-active ufw` only says the service ran. `ufw status`
  is the real check (it's inactive).
- **etcd in k3s speaks gRPC only.** Use `kubectl get --raw /readyz?verbose` for health, and
  `curl -sk -u server:$(sudo cat /var/lib/rancher/k3s/server/token) https://127.0.0.1:6443/db/info`
  on a node for the member list.
- **Flux first install** needs its CRDs before the sync objects, so `50-flux.yml` installs in two steps.
- **k3d on the workstation** (`k3d-thermostat`, host network) creates `flannel.1`/`cni0` on
  this machine with the same pod range (10.42.0.0/24) and VXLAN port as the Pis. It's harmless
  today, but the workstation can't reach Pi pod IPs directly.

## Phase-2 ideas

| Idea | Why |
|---|---|
| **kube-vip** | One floating API IP, so kubectl survives pi1 being down (today it points at pi1) |
| **MetalLB** | Real LoadBalancer IPs from a LAN range instead of ServiceLB's node IPs |
| **Longhorn** | Replicated volumes across nodes (today local-path data lives on one SD card) |
| **cert-manager** | TLS certificates for ingress |
| **kube-prometheus-stack** | Prometheus + Grafana + alerting |
| **SOPS + age** in Flux | Commit encrypted Secrets to the public repo |
| **k3s secrets encryption** | Encrypt Secrets inside etcd, and so inside the snapshots |
| **Tailscale** | Remote access (subnet router for 10.0.0.0/24). Phase 6, skipped for now |
| **Ethernet, NVMe, Ubuntu 26.04** | Each one is a good rebuild exercise |
| **Test restore** on a throwaway VM | Prove `docs/restore.md` works before you need it |
