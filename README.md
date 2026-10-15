# ans-rke2

Builds an [RKE2](https://docs.rke2.io) Kubernetes cluster from VMs that already have RKE2 installed (vm-templates'
`tpl-rocky-10-rke2` template), with [kube-vip](https://kube-vip.io) holding a fixed API address on the servers, and
optionally the add-ons: LoadBalancer addresses for services, a fixed address for RKE2's Traefik ingress,
cert-manager with Let's Encrypt (DNS-01 through Cloudflare), and [Rancher](https://ranchermanager.docs.rancher.com).
Run `site.yml` from AWX or Ascender. Each run brings the cluster in line with the inventory: new hosts join, and
changed settings are applied one node at a time.

| Role | What it does |
| --- | --- |
| [rke2_cluster](roles/rke2_cluster) | Writes each node's `/etc/rancher/rke2/config.yaml`, starts the first server, joins the other servers and the agents one at a time, waits for each node to be Ready, and keeps node labels (including the zone) set |
| [kube_vip](roles/kube_vip) | Runs kube-vip as a static pod on every server; the elected leader answers ARP for the API address, and (with `kube_vip_lb_range`) for LoadBalancer service addresses handed out by kube-vip's cloud provider |
| [rke2_traefik](roles/rke2_traefik) | Gives RKE2's Traefik a fixed LoadBalancer address (`HelmChartConfig`) |
| [cert_manager](roles/cert_manager) | Installs cert-manager and Let's Encrypt `ClusterIssuer`s that solve DNS-01 challenges through Cloudflare |
| [rancher](roles/rancher) | Installs Rancher behind Traefik with a cert-manager certificate and an IP allowlist |
| [kubectl_apply](roles/kubectl_apply) | Helper the others use to `kubectl apply` manifests on a server |

The values committed in `inventory/` are placeholders (`example.com`, `192.0.2.0/24`). Put your real hosts in the
AWX inventory and secrets in AWX credentials, or in local files that are gitignored (`local/`, `extra-vars*.yml`).
Never commit them: this repository is public.

Tested with RKE2 v1.36.5+rke2r1 (Canal, the default Traefik ingress), kube-vip v1.2.4 and ansible-core 2.16 and
2.21, on Rocky Linux 10 nodes in containers: creating a 3-server, 1-agent cluster (with a generated and with a chosen
token), re-running unchanged, setting labels, a run limited to one agent, a run with the first server down, and
kube-vip moving the API address to another server (about 25 seconds). The add-ons (cloud provider v0.0.13,
cert-manager v1.21.2, Rancher 2.15.2) were tested the same way: Traefik on its LoadBalancer address, Rancher answering
through it with its cert-manager certificate, the allowlist returning 403 to other clients, and re-runs unchanged on
both ansible-core versions. The Let's Encrypt/Cloudflare issuance itself was not part of that test.

## Requirements

- The nodes are built from `tpl-rocky-10-rke2` (RKE2 installed but not enabled, no swap, firewalld off), e.g. with
  proxmox-deploy. The play checks for `/usr/bin/rke2` and doesn't install RKE2 itself.
- An unused address on the servers' network for the API (`rke2_cluster_api_address`), outside any DHCP or NetBox
  allocation range (reserve it in NetBox), and ideally a DNS name for it (`rke2_cluster_api_hostname`).
- The nodes can pull images from the internet (RKE2's system images, `ghcr.io/kube-vip/kube-vip`).
- The connection user can `sudo` without a password (vm-templates' `ansible` user can).

## Inventory

| Group | Purpose |
| --- | --- |
| `rke2_servers` | etcd and the control plane. Use 3 (or 5): etcd needs a majority of them up. |
| `rke2_agents` | Workloads only. |

Put both under a parent group (e.g. `rke2_cluster`) and set the cluster-wide variables on it; see
[inventory/hosts.yml](inventory/hosts.yml). Set the variables in the AWX/Ascender inventory or as job extra vars.

With three Proxmox hosts, put one server and one agent on each, and set each host's `rke2_cluster_node_zone` to
its Proxmox node so the cluster can spread replicas across hosts.

### How a run works

1. **Find the cluster.** Every server is checked for etcd data (also servers outside a `--limit`). With none, the
   first server in `rke2_servers` starts a new cluster. Otherwise the server the cluster started on stays the
   "init" server, and the join token is read from it. The play refuses to start a new cluster while any server is
   unreachable, since that server might already hold the cluster.
2. **kube-vip** goes onto every server as a static pod before RKE2 starts.
3. **The init server** gets its configuration and starts. Everyone else joins through
   `https://<rke2_cluster_api_address>:9345`: first the other servers, one at a time, then the agents, one at a
   time. Each node must be Ready before the next one starts.
4. **Labels**: each node's zone and `rke2_cluster_node_labels` are set.

On later runs, a node whose configuration changed restarts RKE2, still one node at a time. A run limited to new
hosts (`--limit new-host`) works: it reads the token from the cluster.

### Adding a node

1. Build the VM from `tpl-rocky-10-rke2`.
2. Add it to `rke2_servers` or `rke2_agents` with its `rke2_cluster_node_zone`.
3. Run the job.

Removing nodes and upgrading RKE2 aren't automated yet.

## Variables

### rke2_cluster

| Variable | Default | Notes |
| --- | --- | --- |
| `rke2_cluster_api_address` | **required** | The API address held by kube-vip. Nodes join through it (9345); clients use it (6443). |
| `rke2_cluster_api_hostname` | `""` | DNS name for that address, added to the API certificate. |
| `rke2_cluster_tls_san` | `[]` | More names/addresses for the API certificate. |
| `rke2_cluster_token` | `""` | Secret. Empty: RKE2 generates the token and the play reads it from the cluster. Set it to choose it; never change it afterwards. |
| `rke2_cluster_cni` | `canal` | Can't be changed once the cluster exists. |
| `rke2_cluster_node_ip` | the default-route address | Per host. Pins the node to its main interface; nodes with storage interfaces need this. |
| `rke2_cluster_node_name` | the host's short name | Per host. |
| `rke2_cluster_node_zone` | `""` | Per host. Sets `topology.kubernetes.io/zone`, e.g. the Proxmox node. |
| `rke2_cluster_node_labels` | `{}` | Per host. Kept set on every run; other labels are left alone. |
| `rke2_cluster_node_taints` | `[]` | Per host, e.g. `["dedicated=storage:NoSchedule"]`. Only applied when the node first registers. |
| `rke2_cluster_server_taint` | `false` | `true` keeps workloads off the servers (`CriticalAddonsOnly=true:NoExecute`), when they first register. |
| `rke2_cluster_server_config` / `_agent_config` | `{}` | More [server](https://docs.rke2.io/reference/server_config) / [agent](https://docs.rke2.io/reference/linux_agent_config) settings for every server / agent. Cluster-wide ones (e.g. `cluster-cidr`) must match on all servers. |
| `rke2_cluster_node_config` | `{}` | Per host, merged last. |
| `rke2_cluster_ready_retries` | `60` | × 10 seconds to wait for the join address and for each node to be Ready. |
| `rke2_cluster_server_group` / `_agent_group` | `rke2_servers` / `rke2_agents` | |

Pod and service networks are RKE2's defaults, `10.42.0.0/16` and `10.43.0.0/16`. Change them before the first run
(`rke2_cluster_server_config: {cluster-cidr: ..., service-cidr: ...}`) if they overlap a network the pods need to
reach.

### kube_vip

| Variable | Default | Notes |
| --- | --- | --- |
| `kube_vip_address` | `rke2_cluster_api_address` | Empty removes kube-vip. |
| `kube_vip_interface` | the default-route interface | Per host. |
| `kube_vip_version` | `v1.2.4` | Image tag of `ghcr.io/kube-vip/kube-vip`. |
| `kube_vip_lease_duration` / `_renew_deadline` / `_retry_period` | `15` / `10` / `2` | Leader election; failover takes about the lease duration. |
| `kube_vip_lb_range` | `""` | Addresses for LoadBalancer services, e.g. `192.0.2.202-192.0.2.220` (or a CIDR). Empty: no cloud provider, services stay pending. |
| `kube_vip_cloud_provider_version` | `v0.0.13` | |

A service can ask for a fixed address with the annotation `kube-vip.io/loadbalancerIPs: <address>`; it doesn't have
to be in the range (Traefik's is outside it).

## Add-ons

The add-ons are installed after the nodes, on one server, each when its variables are set. Each one waits for the
previous: cloud provider → Traefik's address → cert-manager → Rancher. They're installed through RKE2's Helm
controller (`HelmChart` objects in `kube-system`), so the execution environment doesn't need `helm` or `kubectl`.

Turning one off later doesn't uninstall it.

### rke2_traefik

| Variable | Default | Notes |
| --- | --- | --- |
| `rke2_traefik_lb_ip` | `""` | Traefik's fixed address (ports 80/443). Needs `kube_vip_lb_range`. Point the DNS names of the apps it serves here. |
| `rke2_traefik_external_traffic_policy` | `Local` | Keeps client addresses (needed for IP allowlists). |
| `rke2_traefik_values` | `{}` | More values for RKE2's `rke2-traefik` chart. |

RKE2's Traefik is a DaemonSet that also listens on ports 80/443 of every node; that stays as it is.

### cert_manager

| Variable | Default | Notes |
| --- | --- | --- |
| `cert_manager_enabled` | `false` | |
| `cert_manager_acme_email` | **required** | Let's Encrypt account email. |
| `cert_manager_cloudflare_api_token` | **required, secret** | See below. |
| `cert_manager_version` | `v1.21.2` | 1.21 supports Kubernetes 1.33–1.36. |
| `cert_manager_dns01_recursive_nameservers` | `[1.1.1.1:53, 1.0.0.1:53]` | Where cert-manager checks a challenge's TXT record is visible. Public resolvers: internal DNS that hosts the same names (split horizon) never sees the TXT records, and the check would never pass. |
| `cert_manager_issuers` | `letsencrypt-staging`, `letsencrypt-production` | `ClusterIssuer` name → ACME directory URL. |

The Cloudflare token: in Cloudflare, **My Profile → API Tokens → Create Token** with the permissions
**Zone / DNS / Edit** and **Zone / Zone / Read**, limited to the zone(s) certificates are issued for. Give it to the
job through an AWX/Ascender credential (a custom credential type injecting `cert_manager_cloudflare_api_token` as an
extra var), never in the inventory. DNS-01 works for names that exist only in internal DNS: Let's Encrypt only looks
up the `_acme-challenge` TXT record in the public zone.

### rancher

| Variable | Default | Notes |
| --- | --- | --- |
| `rancher_enabled` | `false` | Needs `cert_manager_enabled` and `rke2_traefik_lb_ip`. |
| `rancher_hostname` | **required** | e.g. `rancher.app.example.com`. Its DNS record must point at `rke2_traefik_lb_ip`. |
| `rancher_version` | `2.15.2` | |
| `rancher_replicas` | `3` | |
| `rancher_cluster_issuer` | `letsencrypt-production` | Use `letsencrypt-staging` to test without hitting Let's Encrypt's rate limits. |
| `rancher_allowed_source_ranges` | the private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) | Traefik `ipAllowList`; others get 403. Empty allows everyone. |
| `rancher_bootstrap_password` | `""` | Secret. First-login password; empty lets Rancher generate one. |
| `rancher_values` | `{}` | More chart values. |

Rancher is served on HTTPS only (Traefik's `websecure` entrypoint). To keep it internal, put its DNS record only in
internal DNS; the allowlist also keeps it internal if the router's port-forwards later point at Traefik. The job
ends by printing how to read the first-login password:

```sh
kubectl -n cattle-system get secret bootstrap-secret -o jsonpath={.data.bootstrapPassword} | base64 -d
```

## Using the cluster

On a server, `kubectl` works as root (`sudo -i`, then `kubectl get nodes`): vm-templates puts RKE2's tools on the
PATH and points `KUBECONFIG` at `/etc/rancher/rke2/rke2.yaml`.

For a workstation, copy that file and change its `server:` from `https://127.0.0.1:6443` to
`https://<rke2_cluster_api_hostname>:6443`. It holds cluster-admin credentials: keep it private.

## Troubleshooting

- **A node isn't Ready**: the job prints the node's status. On the node, `journalctl -u rke2-server` (or
  `rke2-agent`) `-n 100`; on a server, `kubectl describe node <name>`.
- **Waiting for the join address times out**: kube-vip isn't holding the address. On a server,
  `crictl ps --name kube-vip` and `crictl logs <id>`. Check `kube_vip_interface` and that nothing else uses the
  address.
- **Traefik doesn't get its address**: check `kube_vip_lb_range` is set and the `kube-vip-cloud-provider` pod in
  `kube-system` is running (its log shows the allocation), and `kubectl -n kube-system describe service rke2-traefik`.
- **Rancher's certificate isn't Ready**: `kubectl -n cattle-system describe certificate tls-rancher-ingress`, then
  `kubectl get challenges -A` and `kubectl describe challenge <name>`: Cloudflare errors (token permissions, wrong
  zone) show there. While testing, use `rancher_cluster_issuer: letsencrypt-staging`.
- **"refuses to start a cluster while servers are unreachable"**: fix the connection to the listed servers, or
  remove them from `rke2_servers`.

## Development and CI

Two workflows call reusable workflows from
[`SAL9000-HomeLab/shared-actions`](https://github.com/SAL9000-HomeLab/shared-actions): **Ansible CI** (yamllint,
syntax check, ansible-lint) and **Linting Validation** (markdownlint, linkspector, yamllint), both on pull requests.

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.16" ansible-lint yamllint
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-lint
ansible-playbook --syntax-check -i inventory/hosts.yml site.yml
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a `vX.Y.Z` tag
publishes that version's section as a GitHub release.
