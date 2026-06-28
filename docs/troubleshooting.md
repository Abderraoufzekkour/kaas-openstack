# Troubleshooting Guide

## CAPI controllers stuck in Pending

**Symptom:** `kubectl get pods -A` shows CAPI pods in `Pending` state.

**Cause:** Control plane node has a `NoSchedule` taint — single-node management cluster cannot schedule pods.

**Fix:**
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

---

## Workload cluster stuck in Provisioning

**Symptom:** `kubectl get cluster kaas-workload` shows `PHASE: Provisioning` for more than 10 minutes.

**Cause:** CAPO cannot reach the OpenStack API, or the cloud credentials are incorrect.

**Fix:**
```bash
# Check CAPO logs
kubectl logs -n capo-system -l control-plane=capo-controller-manager --tail=30

# Verify OpenStack credentials
openstack token issue
```

---

## Workers stuck in Provisioned (not Running)

**Symptom:** `kubectl get machines` shows workers as `Provisioned` but never `Running`.

**Cause:** Workers cannot reach the API server endpoint to complete kubeadm join.

**Fix:** SSH into each worker and run kubeadm join manually using the internal IP of the control plane:
```bash
sudo kubeadm join <CONTROL_PLANE_INTERNAL_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Get the token and hash from the control plane:
```bash
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```

---

## Nodes NotReady after joining

**Symptom:** `kubectl get nodes` shows nodes as `NotReady`.

**Cause:** CNI plugin not installed.

**Fix:**
```bash
kubectl --kubeconfig ~/kaas-workload.kubeconfig apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

---

## Calico pods in ErrImagePull

**Symptom:** `kubectl get pods -n kube-system` shows calico pods with `ErrImagePull`.

**Cause:** Worker nodes cannot pull images from the internet.

**Fix:** Ensure worker nodes have outbound internet access. Verify by SSHing into the worker and running:
```bash
curl -s https://registry.k8s.io
```

---

## ArgoCD applicationset-controller CrashLoopBackOff

**Symptom:** `kubectl get pods -n argocd` shows `argocd-applicationset-controller` in `CrashLoopBackOff`.

**Cause:** The `ApplicationSet` CRD was not installed due to annotation size limit.

**Fix:**
```bash
kubectl --kubeconfig ~/kaas-workload.kubeconfig apply --server-side \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml

kubectl --kubeconfig ~/kaas-workload.kubeconfig delete pod \
  -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller
```
