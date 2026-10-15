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
| [ceph_csi](roles/ceph_csi) | Installs ceph-csi against an existing Ceph cluster (e.g. Proxmox's): RBD block volumes (`ceph-rbd`, the default class) and CephFS shared volumes (`ceph-cephfs`) |
| [nfs_csi](roles/nfs_csi) | Installs csi-driver-nfs with a StorageClass (`nas-nfs`) on an existing NFS share (e.g. a Synology shared folder) |
| [storage_check](roles/storage_check) | Helper the storage roles use: provisions a test volume and writes to it from a pod |
| [cert_manager](roles/cert_manager) | Installs cert-manager and Let's Encrypt `ClusterIssuer`s that solve DNS-01 challenges through Cloudflare |
| [rancher](roles/rancher) | Installs Rancher behind Traefik with a cert-manager certificate and an IP allowlist |
| [fleet_bootstrap](roles/fleet_bootstrap) | Points Rancher's Fleet at the platform GitOps repository (k8s-platform) with a read-only GitHub token |
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
both ansible-core versions. The Let's Encrypt/Cloudflare issuance itself was not part of that test. The storage
add-ons were tested against a single-node Ceph 19.2.7 (aes256k keys, users created as in "Preparing Ceph") and a kernel
NFS server: each StorageClass's check passed (RBD through `rbd-nbd`, CephFS through `ceph-fuse`, NFS archived on
delete).

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

## Storage

Three StorageClasses, each from storage outside the cluster:

| StorageClass | Driver | Access | Use it for |
| --- | --- | --- | --- |
| `ceph-rbd` (default) | ceph-csi RBD | ReadWriteOnce | Databases and anything one pod uses: replicated across the Proxmox hosts, fast. |
| `ceph-cephfs` | ceph-csi CephFS | ReadWriteMany | Volumes several pods share, highly available. |
| `nas-nfs` | csi-driver-nfs | ReadWriteMany | Bulk and shared data on the NAS. The NAS is a single point of failure, and databases do badly on NFS. |

A claim that names no StorageClass gets `ceph-rbd`. After installing each class, the job provisions a 1 GiB test
volume, mounts it in a pod and writes to it, then deletes it, so a wrong key, address or share permission fails the
job instead of the first app (turn it off with `ceph_csi_check` / `nfs_csi_check`).

The nodes reach Ceph and the NAS directly: on Proxmox, give every node a NIC on the Ceph public network and one on the
NAS storage network (proxmox-deploy's `extra_nics`). Pods' traffic to them leaves through those NICs.

**Userspace Ceph clients.** Current Ceph releases give new users **aes256k** keys, the fix for CVE-2025-30156 (an
authentication bypass in CephX's old AES cipher). The kernel's RBD and CephFS clients only support aes256k from
Linux 7.0, and Rocky 10's kernel is 6.12 (it logs `libceph: secret too big 32` and refuses the key). So the nodes use
the userspace clients, which do: `rbd-nbd` for RBD and `ceph-fuse` for CephFS (`ceph_csi_rbd_mounter`,
`ceph_csi_cephfs_mounter`). Don't work around it by creating the users with `--key-type=aes`: that needs the old
cipher allowed for authentication across the whole Ceph cluster (`auth_allowed_ciphers`), which keeps the CVE open
for Proxmox too. Switch to `krbd` / `kernel` once the nodes' kernel supports aes256k.

The userspace clients run inside the ceph-csi node plugin pods, so restarting a node plugin (an upgrade of ceph-csi,
or a changed chart value) cuts the volumes mounted on that node. Drain a node before anything restarts its node
plugins (`kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`, then `kubectl uncordon <node>`). ceph-csi
reattaches `rbd-nbd` volumes when its node plugin restarts; CephFS volumes need their pods restarted.

### Preparing Ceph (Proxmox)

Run these once, as root on any Proxmox node (`pveceph` is Proxmox's wrapper around the Ceph commands). The names
(`k8s-rbd`, `cephfs`, `k8s-rbd` / `k8s-cephfs` users) are the role defaults; use your own consistently.

1. **RBD pool** for block volumes (replicated 3/2 by default):

   ```sh
   pveceph pool create k8s-rbd --application rbd --add_storages 0
   ```

   `--add_storages 0` keeps it out of Proxmox's own storage list, so VM disks don't land in it.

2. **CephFS** for shared volumes. Skip the first two commands if the cluster already has a CephFS (`ceph fs ls`) and
   use its name. A metadata server (MDS) on each node gives one active and two standbys:

   ```sh
   pveceph mds create            # run on each Proxmox node
   pveceph fs create --name cephfs --add-storage 0
   ceph fs subvolumegroup create cephfs csi
   ```

   ceph-csi puts each volume in a subvolume of the group `csi` (`ceph_csi_cephfs_subvolume_group`), and since
   ceph-csi 3.10 it no longer creates the group itself.

3. **Ceph users** for Kubernetes, each limited to its pool or filesystem
   ([ceph-csi capabilities](https://github.com/ceph/ceph-csi/blob/devel/docs/capabilities.md)):

   ```sh
   ceph auth get-or-create client.k8s-rbd \
     mon 'profile rbd' \
     osd 'profile rbd pool=k8s-rbd' \
     mgr 'profile rbd pool=k8s-rbd'

   ceph auth get-or-create client.k8s-cephfs \
     mgr 'allow rw' \
     osd 'allow rw tag cephfs metadata=cephfs, allow rw tag cephfs data=cephfs' \
     mds 'allow r fsname=cephfs path=/volumes, allow rws fsname=cephfs path=/volumes/csi' \
     mon 'allow r fsname=cephfs'
   ```

   (With a different filesystem name, replace every `cephfs` after `metadata=`, `data=` and `fsname=`.) Create them
   without `--key-type`: the default aes256k key is what the userspace clients need (see above).

4. **The values the job needs**:

   ```sh
   ceph fsid                          # -> ceph_csi_cluster_id
   ceph mon dump | grep -o 'v1:[0-9.]*:6789' | cut -c4-   # -> ceph_csi_monitors
   ceph auth get-key client.k8s-rbd; echo     # -> ceph_csi_rbd_key (secret)
   ceph auth get-key client.k8s-cephfs; echo  # -> ceph_csi_cephfs_key (secret)
   ```

   The monitor addresses are on the Ceph public network (`public_network` in `/etc/pve/ceph.conf`).

### Preparing the NFS share (Synology DSM 7)

1. **Control Panel → File Services → NFS**: tick **Enable NFS service**, set **Maximum NFS protocol** to
   **NFSv4.1**, and apply.
2. **Control Panel → Shared Folder → Create**: e.g. `k8s` on the volume you want. Leave encryption off; the recycle
   bin is optional (the driver archives deleted volumes anyway, see `nfs_csi_on_delete`).
3. Select the folder, **Edit → NFS Permissions → Create**:

   | Setting | Value | Why |
   | --- | --- | --- |
   | Hostname or IP | the nodes' storage network, e.g. `192.0.2.0/24` | Only the cluster nodes mount it. |
   | Privilege | Read/Write | |
   | Squash | **No mapping** | Containers run as root and change ownership (e.g. a database's data directory); squashing root breaks them. |
   | Security | sys | |
   | Enable asynchronous | ticked | Faster writes; DSM's default. |
   | Allow connections from non-privileged ports | unticked | Linux clients use privileged ports. |
   | Allow users to access mounted subfolders | **ticked** | Each volume is a subfolder that nodes mount on its own. |

4. The dialog's footer shows the **Mount path**, e.g. `/volume1/k8s`: that's `nfs_csi_share`. `nfs_csi_server` is
   the NAS's address on the storage network the nodes reach (not its management address).

Snapshots of the folder (Snapshot Replication) and Hyper Backup cover the volumes on it, as for any shared folder.

### The Ceph keys in AWX/Ascender

Like the Cloudflare token, the Ceph keys come from a credential. **Administration → Credential Types → Add**, name it
`Ceph keys (ceph-csi)`:

```yaml
# Input configuration
fields:
  - id: ceph_csi_rbd_key
    label: Key of the RBD user (ceph auth get-key client.k8s-rbd)
    type: string
    secret: true
  - id: ceph_csi_cephfs_key
    label: Key of the CephFS user (ceph auth get-key client.k8s-cephfs)
    type: string
    secret: true
```

```yaml
# Injector configuration
extra_vars:
  ceph_csi_rbd_key: "{{ ceph_csi_rbd_key }}"
  ceph_csi_cephfs_key: "{{ ceph_csi_cephfs_key }}"
```

Create a credential of that type with both keys and attach it to the job template. The job stores them as the
Secrets `csi-rbd-secret` (namespace `ceph-csi-rbd`) and `csi-cephfs-secret` (`ceph-csi-cephfs`).

### ceph_csi

| Variable | Default | Notes |
| --- | --- | --- |
| `ceph_csi_enabled` | `false` | |
| `ceph_csi_cluster_id` / `ceph_csi_monitors` | **required** | `ceph fsid`; monitor addresses, e.g. `["192.0.2.11:6789", "192.0.2.12:6789", "192.0.2.13:6789"]`. |
| `ceph_csi_rbd_pool` | `""` | Empty: no RBD driver. |
| `ceph_csi_rbd_user` / `ceph_csi_rbd_key` | `k8s-rbd` / **secret** | The user without `client.`. |
| `ceph_csi_rbd_storage_class` / `_default_class` | `ceph-rbd` / `true` | |
| `ceph_csi_rbd_fstype` / `_image_features` | `ext4` / `layering,exclusive-lock,object-map,fast-diff,deep-flatten` | |
| `ceph_csi_rbd_mounter` | `rbd-nbd` | `krbd` once the nodes' kernel supports aes256k keys (Linux 7.0+). |
| `ceph_csi_cephfs_fs_name` | `""` | Empty: no CephFS driver. |
| `ceph_csi_cephfs_subvolume_group` | `csi` | Must exist. |
| `ceph_csi_cephfs_user` / `ceph_csi_cephfs_key` | `k8s-cephfs` / **secret** | |
| `ceph_csi_cephfs_storage_class` | `ceph-cephfs` | |
| `ceph_csi_cephfs_mounter` | `fuse` | `kernel` once the nodes' kernel supports aes256k keys. |
| `ceph_csi_rbd_metrics_port` / `_cephfs_metrics_port` | `8080` / `8082` | Host ports of the node plugins' metrics. Not 8081: Canal's flanneld listens there on RKE2. |
| `ceph_csi_reclaim_policy` | `Delete` | `Retain` keeps a volume's data after its claim is deleted. |
| `ceph_csi_version` | `3.18.1` | |
| `ceph_csi_check` | `true` | |

### nfs_csi

| Variable | Default | Notes |
| --- | --- | --- |
| `nfs_csi_enabled` | `false` | |
| `nfs_csi_server` / `nfs_csi_share` | **required** | e.g. `192.0.2.20` / `/volume1/k8s`. |
| `nfs_csi_storage_class` | `nas-nfs` | |
| `nfs_csi_sub_dir` | `<namespace>-<claim>-<volume>` | Each volume's folder on the share. |
| `nfs_csi_on_delete` | `archive` | When a claim is deleted: `archive` renames its folder `archived-…`, `retain` leaves it, `delete` removes it. |
| `nfs_csi_mount_options` | `[nfsvers=4.1, hard]` | |
| `nfs_csi_controller_replicas` | `2` | |
| `nfs_csi_version` | `4.13.5` | |
| `nfs_csi_check` | `true` | |

A StorageClass's parameters can't be changed once it exists: to change the share or options, delete the
StorageClass (`kubectl delete storageclass nas-nfs`; existing volumes keep working) and run the job again.

## Add-ons

The add-ons are installed after the nodes, on one server, each when its variables are set. Each one waits for the
previous: cloud provider → Traefik's address → Ceph storage → NFS storage → cert-manager → Rancher → Fleet. They're
installed
through RKE2's Helm controller (`HelmChart` objects in `kube-system`), so the execution environment doesn't need `helm`
or `kubectl`.

Turning one off later doesn't uninstall it.

`kubectl get cluster` on this cluster means Rancher's Cluster API type; CloudNativePG's is
`clusters.postgresql.cnpg.io`.

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
| `cert_manager_cloudflare_api_token` | **required, secret** | From an AWX/Ascender credential: see [The Cloudflare API token](#the-cloudflare-api-token). |
| `cert_manager_version` | `v1.21.2` | 1.21 supports Kubernetes 1.33–1.36. |
| `cert_manager_dns01_recursive_nameservers` | `[1.1.1.1:53, 1.0.0.1:53]` | Where cert-manager checks a challenge's TXT record is visible. Public resolvers: internal DNS that hosts the same names (split horizon) never sees the TXT records, and the check would never pass. |
| `cert_manager_issuers` | `letsencrypt-staging`, `letsencrypt-production` | `ClusterIssuer` name → ACME directory URL. |

#### The Cloudflare API token

1. In Cloudflare, under **My Profile → API Tokens → Create Token**, create a custom token with the permissions
   **Zone / DNS / Edit** and **Zone / Zone / Read**, and under **Zone Resources** include only the zone(s)
   certificates are issued for (e.g. `example.com`).
2. Give it to the job through an AWX/Ascender credential, never in the inventory or the repository. Under
   **Administration → Credential Types → Add**, name it `Cloudflare API token (cert-manager)` and paste:

   ```yaml
   # Input configuration
   fields:
     - id: cert_manager_cloudflare_api_token
       label: Cloudflare API token
       type: string
       secret: true
       help_text: Zone / DNS / Edit and Zone / Zone / Read on the zones certificates are issued for
   required:
     - cert_manager_cloudflare_api_token
   ```

   ```yaml
   # Injector configuration
   extra_vars:
     cert_manager_cloudflare_api_token: "{{ cert_manager_cloudflare_api_token }}"
   ```

3. Under **Resources → Credentials → Add**, create a credential of type `Cloudflare API token (cert-manager)` with
   the token.
4. Attach it to the `site.yml` job template, next to the Machine credential (the SSH key for the nodes).

The job stores the token in the cluster as the Secret `cloudflare-api-token` in the `cert-manager` namespace and
updates it when the credential changes; it never appears in the job log. DNS-01 works for names that exist only in
internal DNS: Let's Encrypt only looks up the `_acme-challenge` TXT record in the public zone.

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

### fleet_bootstrap

Rancher includes [Fleet](https://fleet.rancher.io), its GitOps engine. This role gives it one repository to deploy, the
platform repository (k8s-platform), which in turn lists the app repositories. Everything after that is deployed from
Git, not by this job.

| Variable | Default | Notes |
| --- | --- | --- |
| `fleet_bootstrap_enabled` | `false` | Needs `rancher_enabled`. |
| `fleet_bootstrap_repo` | **required** | HTTPS URL, e.g. `https://github.com/SAL9000-HomeLab/k8s-platform`. |
| `fleet_bootstrap_branch` | `main` | |
| `fleet_bootstrap_name` | `k8s-platform` | The GitRepo's name; bundle names start with it. |
| `fleet_bootstrap_github_token` | `""` | Secret. A read-only GitHub token for private repositories; empty for a public one. |
| `fleet_bootstrap_secret_name` | `fleet-github` | The Secret in `fleet-local` holding the token; the app GitRepos refer to it. |

The job checks that Fleet cloned the repository and fails with Fleet's message if it didn't (a wrong token, or a
repository the token can't read). The GitRepo scans each top-level directory with a `fleet.yaml` (`paths: ["*"]`):
without that, Fleet also makes the repository's root a bundle, which takes in the subdirectories' files too.

**The GitHub token:** a fine-grained personal access token (GitHub → **Settings → Developer settings → Personal access
tokens → Fine-grained tokens**): resource owner `SAL9000-HomeLab`, **Only select repositories** (k8s-platform and the
`k8s-*` app repositories), permission **Contents: Read-only**. Add new app repositories to it as they're created. Give
it to the job through a credential type:

```yaml
# Input configuration
fields:
  - id: fleet_bootstrap_github_token
    label: GitHub token (Fleet, read-only)
    type: string
    secret: true
required:
  - fleet_bootstrap_github_token
```

```yaml
# Injector configuration
extra_vars:
  fleet_bootstrap_github_token: "{{ fleet_bootstrap_github_token }}"
```

When the token expires, Fleet stops picking up changes (what's deployed keeps running): put the new one in the
credential and run the job.

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
- **The storage check fails** (`couldn't be provisioned and written`): the job prints the claim's and pod's events.
  - `ceph-rbd` / `ceph-cephfs` claim stays Pending: the provisioner can't reach or authenticate to Ceph. Check
    `ceph_csi_monitors` (Ceph public network addresses, reachable from the nodes), the key, and the user's
    capabilities (`ceph auth get client.k8s-rbd`); the provisioner's log:
    `kubectl -n ceph-csi-rbd logs deploy/ceph-csi-rbd-provisioner -c csi-provisioner`.
  - CephFS: `subvolume group 'csi' does not exist` → `ceph fs subvolumegroup create cephfs csi`.
  - The pod can't mount and the node's kernel log (`dmesg`) says `libceph: secret too big 32`: a kernel client
    (`krbd` / `kernel` mounter) got an aes256k key. Use `rbd-nbd` / `fuse` (the defaults).
  - `AttachVolume.Attach failed … CSINode … does not contain driver`: the driver's node plugin isn't running on that
    node (`kubectl -n ceph-csi-cephfs get pods -o wide`); a node plugin stuck Pending with "didn't have free ports"
    has a metrics port that something else on the node uses.
  - The pod can't mount (`MountVolume` errors): the node itself can't reach Ceph or the NAS on its storage NIC.
  - `nas-nfs`: `access denied` → the NFS rule doesn't include the nodes' storage network; `permission denied`
    writing → Squash isn't **No mapping**; mounting a subfolder fails → tick **Allow users to access mounted
    subfolders**.
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
