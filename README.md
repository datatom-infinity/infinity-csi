# Infinitycsi

This repo contains DATATOM Infinity Storage
[Container Storage Interface (CSI)](https://github.com/container-storage-interface/)
driver for block, fs and kubernetes sidecar deployment yamls of provisioner,
attacher, node-driver-registrar and snapshotter for supporting CSI functionalities.

## Overview

InfinityCSI plugins implement an interface between CSI enabled Container Orchestrator
(CO) and infinity cluster. It allows dynamically provisioning infinity volumes and
attaching them to workloads.

Independent CSI plugins are provided to support block and fs backed volumes,

For details about configuration and deployment of block plugin, please refer
  [block doc](./docs/deploy-block.md) and
  for fs plugin configuration and deployment please
  refer [fs doc](./docs/deploy-fs.md).
