# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Added: `site.yml` with the `rke2_cluster` and `kube_vip` roles: builds an RKE2 cluster from `rke2_servers` and
  `rke2_agents` on hosts with RKE2 installed (vm-templates' `tpl-rocky-10-rke2`). Finds an existing cluster before
  doing anything (and refuses to start a second one while a server is unreachable), starts the first server, joins
  the other servers and the agents one at a time through the kube-vip API address, waits for each node to be Ready,
  pins each node to its default-route address, and keeps zone and node labels set. The join token is read from the
  cluster unless `rke2_cluster_token` is set.
