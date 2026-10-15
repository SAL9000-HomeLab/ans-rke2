# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Added: storage add-ons. `ceph_csi` installs ceph-csi 3.18 against an existing Ceph cluster: RBD (`ceph-rbd`, the
  default StorageClass) and CephFS (`ceph-cephfs`), each with its key in a Secret, through the userspace clients
  (`rbd-nbd`, `ceph-fuse`) because Ceph's aes256k keys need Linux 7.0 for the kernel clients. `nfs_csi` installs csi-driver-nfs
  4.13 with a `nas-nfs` StorageClass on an NFS share (deleted volumes archived). After installing, each StorageClass
  is checked with a test volume a pod writes to (`storage_check`). The README documents preparing Ceph on Proxmox, the
  Synology NFS share, and the AWX/Ascender credential type for the Ceph keys.
- Added: cluster add-ons, each switched on by its variables: kube-vip LoadBalancer services through kube-vip's cloud
  provider (`kube_vip_lb_range`), a fixed LoadBalancer address for RKE2's Traefik (`rke2_traefik` role), cert-manager
  v1.21 with Let's Encrypt issuers using Cloudflare DNS-01 (`cert_manager`), and Rancher 2.15 behind Traefik with a
  cert-manager certificate and an IP allowlist (`rancher`). The kube-vip play now updates one server at a time.
  The README describes the AWX/Ascender credential type for the Cloudflare API token.
- Added: `site.yml` with the `rke2_cluster` and `kube_vip` roles: builds an RKE2 cluster from `rke2_servers` and
  `rke2_agents` on hosts with RKE2 installed (vm-templates' `tpl-rocky-10-rke2`). Finds an existing cluster before
  doing anything (and refuses to start a second one while a server is unreachable), starts the first server, joins
  the other servers and the agents one at a time through the kube-vip API address, waits for each node to be Ready,
  pins each node to its default-route address, and keeps zone and node labels set. The join token is read from the
  cluster unless `rke2_cluster_token` is set.
