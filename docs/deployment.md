# Deployment Guide

## Prerequisites

- OpenStack project with API access
- Ubuntu 22.04 VM (bastion) with floating IP
- All tools installed (kubectl, kind, terraform, clusterctl, helm, docker, git)

## Step 1 - Prepare OpenStack credentials

Download your clouds.yaml from OpenStack Horizon:
Project → API Access → Download OpenStack clouds.yaml File

Add your password to the auth section and place it at:
~/.config/openstack/clouds.yaml

## Step 2 - Bootstrap the management cluster

```bash
kind create cluster --name kaas-mgmt --wait 5m
```

## Step 3 - Install Cluster API + CAPO

```bash
export OPENSTACK_CLOUD_YAML_B64=$(base64 -w0 ~/.config/openstack/clouds.yaml)
export OPENSTACK_CLOUD=openstack
export OPENSTACK_CLOUD_CACERT_B64=""
clusterctl init --infrastructure openstack:v0.10.4
```

## Note - Air-gapped image loading

If your environment restricts outbound access from Docker containers,
pull images on the bastion and load them manually into kind:

```bash
docker pull --platform linux/amd64 <image>
docker save <image> | docker exec -i kaas-mgmt-control-plane \
  ctr --namespace=k8s.io images import --snapshotter=overlayfs -
```
