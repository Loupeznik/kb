# RKE2 Cluster Troubleshooting Guide

This document covers common issues encountered with RKE2 clusters and their solutions.

Issues encountered here are based on deployment of multi-node RKE2 clusters from the [RKE2 Ansible Playbook](https://github.com/Loupeznik/playbooks/tree/master/debian_based/rke2-cluster) deployed on Proxmox and bootstrapped via the Rachner dashboard.

## Table of Contents

- [Calico Networking Issues](#calico-networking-issues)
- [Longhorn Storage Issues](#longhorn-storage-issues)
- [CSI Pod Exec Format Errors](#csi-pod-exec-format-errors)
- [General Troubleshooting Commands](#general-troubleshooting-commands)

## Calico Networking Issues

### Symptom: Calico Node Pod Not Ready

**Symptoms:**
- Calico node pod shows 0/1 Ready status
- Pods on affected node cannot communicate with cluster services
- Longhorn manager and other network-dependent services fail

**Example:**
```bash
kubectl get pods -n calico-system
NAME                READY   STATUS    RESTARTS   AGE
calico-node-64rz2   0/1     Running   0          22h
```

**Diagnosis:**

Check Calico node pod logs:
```bash
kubectl logs -n calico-system <calico-node-pod-name>
```

Look for errors like:
```
Failed to connect to typha endpoint 10.9.11.46:5473... i/o timeout
```

**Root Cause:**

Port 5473 (Calico Typha) is not allowed through the firewall. Calico uses Typha as a connection concentrator for the Kubernetes API, and without this port open, Calico nodes cannot establish proper communication.

**Solution:**

Add port 5473/tcp to the firewall on all nodes:

```bash
sudo ufw allow 5473/tcp comment 'Calico Typha'
```

Verify the rule was added:
```bash
sudo ufw status numbered | grep 5473
```

Restart the affected Calico pod:
```bash
kubectl delete pod -n calico-system <calico-node-pod-name>
```

Verify the new pod becomes Ready:
```bash
kubectl get pods -n calico-system -w
```

**Prevention:**

Ensure the playbook's `group_vars/all.yml` includes port 5473 in both `controlplane_ports` and `worker_ports`.

## Longhorn Storage Issues

### Symptom: Longhorn Manager Crashes

**Symptoms:**
- Longhorn manager pods in CrashLoopBackOff
- PVCs stuck in Pending state
- Longhorn admission webhook failures

**Diagnosis:**

Check longhorn-manager pod logs:
```bash
kubectl logs -n longhorn-system <longhorn-manager-pod-name>
```

Look for errors like:
```
Failed to initialize Longhorn API client
connection timed out
```

Check Longhorn backend service endpoints:
```bash
kubectl get endpoints -n longhorn-system longhorn-backend
```

**Root Cause:**

Typically caused by networking issues (see Calico section above). Longhorn manager requires functional cluster networking to communicate with the Longhorn backend service.

**Solution:**

1. Fix underlying networking issues first (ensure Calico is healthy)
2. Delete the crashing longhorn-manager pod to force restart:
```bash
kubectl delete pod -n longhorn-system <longhorn-manager-pod-name>
```

3. Verify all nodes are registered in Longhorn:
```bash
kubectl get nodes.longhorn.io -n longhorn-system
```

Expected output shows all cluster nodes:
```
NAME        READY   ALLOWSCHEDULING   SCHEDULABLE   AGE
master-01   True    true              True          22h
master-02   True    true              True          19m
worker-01   True    true              True          22h
```

### Symptom: PVC Stuck in Pending

**Diagnosis:**

Check PVC status:
```bash
kubectl get pvc -n <namespace>
kubectl describe pvc -n <namespace> <pvc-name>
```

Check Longhorn CSI provisioner pods:
```bash
kubectl get pods -n longhorn-system -l app=csi-provisioner
```

**Solution:**

Ensure all CSI components are running (see CSI Pod Exec Format Errors section if they're not).

## CSI Pod Exec Format Errors

### Symptom: CSI Pods CrashLoopBackOff with Exec Format Error

**Symptoms:**
- CSI pods (attacher, provisioner, resizer, snapshotter) in CrashLoopBackOff
- longhorn-csi-plugin pods not ready (0/3 or similar)
- Error in logs: `exec /csi-attacher: exec format error`

**Example:**
```bash
kubectl get pods -n longhorn-system | grep csi
csi-attacher-858dd64dc4-6w768       0/1     CrashLoopBackOff   9 (2m37s ago)   23m
csi-provisioner-7d9f559dcd-dmvgv    0/1     CrashLoopBackOff   265 (43s ago)   21h
```

**Diagnosis:**

Check pod logs:
```bash
kubectl logs -n longhorn-system <csi-pod-name>
```

Error message:
```
exec /csi-attacher: exec format error
```

This indicates corrupted or incorrectly cached container images.

**Root Cause:**

The exec format error occurs when container images are corrupted or cached with the wrong architecture binaries. This can happen during image pulls or due to registry issues.

**Solution:**

1. Identify which nodes have failing CSI pods:
```bash
kubectl get pods -n longhorn-system -l app=csi-attacher -o wide
kubectl get pods -n longhorn-system -l app=csi-provisioner -o wide
kubectl get pods -n longhorn-system -l app=csi-resizer -o wide
kubectl get pods -n longhorn-system -l app=csi-snapshotter -o wide
```

2. SSH to the affected node and remove the corrupted images:

For CSI attacher:
```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl images | grep csi-attacher

sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl rmi <image-id>
```

For all CSI images at once:
```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl rmi $(sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl images | grep -E 'csi-(attacher|provisioner|resizer|snapshotter)' | awk '{print $3}' | sort -u)
```

3. Delete the failing pods to force recreation with fresh images:
```bash
kubectl get pods -n longhorn-system -l app=csi-attacher -o wide | grep <node-name> | awk '{print $1}' | \
  xargs kubectl delete pod -n longhorn-system
```

Repeat for provisioner, resizer, and snapshotter.

4. Verify new pods are Running:
```bash
kubectl get pods -n longhorn-system | grep csi
```

Expected output:
```
csi-attacher-858dd64dc4-dr9x5       1/1     Running   0          2m
csi-provisioner-7d9f559dcd-4j5x4    1/1     Running   0          1m
csi-resizer-567484d9f7-t828z        1/1     Running   0          1m
csi-snapshotter-858f58fdc5-4hj86    1/1     Running   0          1m
```

### Symptom: longhorn-csi-plugin Pod Not Ready

**Diagnosis:**

Check pod status:
```bash
kubectl get pods -n longhorn-system -l app=longhorn-csi-plugin
```

Check individual container status:
```bash
kubectl describe pod -n longhorn-system <longhorn-csi-plugin-pod-name>
```

Check logs for each container:
```bash
kubectl logs -n longhorn-system <pod-name> -c node-driver-registrar
kubectl logs -n longhorn-system <pod-name> -c livenessprobe
kubectl logs -n longhorn-system <pod-name> -c longhorn-csi-plugin
```

**Solution:**

Same as CSI pod exec format errors. Remove the corrupted images and restart the pods:

```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl rmi $(sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl images | grep -E 'csi-node-driver-registrar|livenessprobe' | awk '{print $3}' | sort -u)

kubectl delete pod -n longhorn-system <longhorn-csi-plugin-pod-name>
```

## General Troubleshooting Commands

### Cluster Health

Check node status:
```bash
kubectl get nodes -o wide
```

Check all pods across namespaces:
```bash
kubectl get pods -A
```

Check pods not running:
```bash
kubectl get pods -A | grep -v Running | grep -v Completed
```

### Calico Troubleshooting

List all Calico pods:
```bash
kubectl get pods -n calico-system
```

Check Calico node status on specific node:
```bash
kubectl logs -n calico-system <calico-node-pod-name>
```

### Longhorn Troubleshooting

Check all Longhorn pods:
```bash
kubectl get pods -n longhorn-system
```

Check Longhorn nodes:
```bash
kubectl get nodes.longhorn.io -n longhorn-system
```

Check Longhorn volumes:
```bash
kubectl get volumes.longhorn.io -n longhorn-system
```

Check Longhorn backend endpoints:
```bash
kubectl get endpoints -n longhorn-system longhorn-backend
```

Check PVCs using Longhorn:
```bash
kubectl get pvc -A | grep longhorn
```

### Container Runtime Commands

List images on a node:
```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl images
```

List containers on a node:
```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl ps -a
```

Remove an image:
```bash
ssh ubuntu@<node-ip> -i ~/.ssh/<key>
sudo CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock \
  /var/lib/rancher/rke2/bin/crictl rmi <image-id>
```

### Firewall Troubleshooting

Check UFW status:
```bash
sudo ufw status numbered
```

Add a firewall rule:
```bash
sudo ufw allow <port>/<protocol> comment '<description>'
```

Reload UFW:
```bash
sudo ufw reload
```

### Log Analysis

Get recent pod logs:
```bash
kubectl logs -n <namespace> <pod-name> --tail=100
```

Get previous pod logs (after crash):
```bash
kubectl logs -n <namespace> <pod-name> --previous
```

Stream pod logs:
```bash
kubectl logs -n <namespace> <pod-name> -f
```

Get logs from specific container in multi-container pod:
```bash
kubectl logs -n <namespace> <pod-name> -c <container-name>
```

## Common Issue Resolution Checklist

When troubleshooting RKE2 cluster issues, follow this checklist:

1. **Check node status** - Ensure all nodes are Ready
2. **Check networking** - Verify Calico pods are Running and Ready
3. **Check firewall** - Ensure all required ports are open (especially 5473 for Calico)
4. **Check storage** - Verify Longhorn manager pods are Running
5. **Check CSI** - Ensure all CSI component pods are Running
6. **Check PVCs** - Verify PVCs are Bound if storage is required
7. **Check logs** - Review pod logs for specific error messages

## Known Issues and Workarounds

### Issue: Node hostname resolution errors

**Symptom:**
```
sudo: unable to resolve host <hostname>: Name or service not known
```

**Cause:**
The /etc/hosts file has stale or incorrect entries for the node's hostname.

**Solution:**
Already handled by the playbook. The playbook ensures /etc/hosts has the correct entry:
```
127.0.1.1 <hostname>
```

### Issue: Cloud provider taint preventing pod scheduling

**Symptom:**
Pods stuck in Pending state with taint-related errors.

**Cause:**
RKE2 may add a cloud provider taint even when not using a cloud provider.

**Solution:**
Remove the taint:
```bash
kubectl taint nodes <node-name> node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```
