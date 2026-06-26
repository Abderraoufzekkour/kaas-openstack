# Deployment Guide

## Step 1 - Bootstrap Management Cluster

### Create kind cluster
```bash
kind create cluster --name kaas-mgmt --wait 5m
```

### Install CAPI + CAPO
```bash
export OPENSTACK_CLOUD_YAML_B64=$(base64 -w0 ~/.config/openstack/clouds.yaml)
export OPENSTACK_CLOUD=openstack
export OPENSTACK_CLOUD_CACERT_B64=""
clusterctl init --infrastructure openstack:v0.10.4
```

### Note on air-gapped image loading
ICOSNET restricts Docker network access from kind containers.
Images must be pulled on the bastion and loaded manually:
```bash
docker save <image> | docker exec -i kaas-mgmt-control-plane ctr --namespace=k8s.io images import --snapshotter=overlayfs -
```
