# GKE E2E Testing for Filestore CSI Share Pools

This guide walks you through the step-by-step process of compiling, configuring, deploying, and testing the Filestore CSI driver's Share Pools feature on a GKE cluster.

---

## Prerequisites & Setup

Before starting, define your environment variables to match your GCP project and cluster settings:

```bash
# Define your target Google Container Registry (GCR) project path
export GCR_PROJECT="gcr.io/ayug-consumer"
export DRIVER_IMAGE="${GCR_PROJECT}/gcp-filestore-csi-driver"
export IMAGE_TAG="sharepools-test-v2"

# Ensure you are connected to the correct GKE cluster context
kubectl config current-context
```

---

## Step 1: Build the Driver Image
Run the docker build command from the root of the `gcp-filestore-csi-driver` repository:

```bash
docker build -t ${DRIVER_IMAGE}:${IMAGE_TAG} .
```

---

## Step 2: Push the Image to GCR
Authenticate with Google Cloud and push the image to GCR so your GKE cluster can access it:

```bash
docker push ${DRIVER_IMAGE}:${IMAGE_TAG}
```

---

## Step 3: Configure Kustomize Overlays

The open-source driver manifests expect self-managed credentials. On GKE, we configure the manifests to use native GKE Workload Identity. 

Navigate to the overlay folder `deploy/kubernetes/overlays/sharepool/` and configure the following files:

### 1. `kustomization.yaml`
This file instructs Kustomize to swap the default container image with your custom image and strips out unneeded local secret mounts:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../stable-master
patchesStrategicMerge:
- controller_patch.yaml
- node_patch.yaml
namespace: gcp-filestore-csi-driver

# Point Kustomize to pull your newly pushed GCR image
images:
- name: registry.k8s.io/cloud-provider-gcp/gcp-filestore-csi-driver
  newName: gcr.io/ayug-consumer/gcp-filestore-csi-driver
  newTag: sharepools-test-v2

# Remove local secrets & credentials mounts (GKE handles this via Workload Identity)
patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: gcp-filestore-csi-controller
    namespace: gcp-filestore-csi-driver
  patch: |
    - op: remove
      path: /spec/template/spec/volumes/1
    - op: remove
      path: /spec/template/spec/containers/0/volumeMounts/1
    - op: remove
      path: /spec/template/spec/containers/0/env
```

### 2. `controller_patch.yaml`
This configuration patches the controller deployment, routes requests to the Filestore Autopush sandbox API, and supplies the necessary GCR registry authentication key:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gcp-filestore-csi-controller
  namespace: gcp-filestore-csi-driver
spec:
  template:
    spec:
      imagePullSecrets:
      - name: gcr-json-key
      containers:
      - name: gcp-filestore-driver
        args:
        - "--v=4"
        - "--endpoint=unix:/csi/csi.sock"
        - "--nodeid=$(KUBE_NODE_NAME)"
        - "--controller=true"
        - "--feature-share-pools=true"
        - "--primary-filestore-service-endpoint=https://test-file.sandbox.googleapis.com"
```

### 3. `node_patch.yaml`
This patch configures the node DaemonSet to pull the driver image successfully using your registry credentials:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gcp-filestore-csi-node
  namespace: gcp-filestore-csi-driver
spec:
  template:
    spec:
      imagePullSecrets:
      - name: gcr-json-key
```

---

## Step 4: Deploy the Driver to GKE
Apply the overlay configuration using the Kustomize directive `-k`:

```bash
kubectl apply -k deploy/kubernetes/overlays/sharepool
```

### Checking Driver Health:
Wait ~30 seconds and run the following command. Ensure all controller and node pods are `Running` healthily:

```bash
kubectl get pods -n gcp-filestore-csi-driver
```

---

## Step 5: Run the Share Pool Test

### 1. Create the StorageClass & PVC manifest
Use the example manifests provided in the codebase:
```bash
# Apply StorageClass and PVC example manifests
kubectl apply -f examples/kubernetes/sharepool/demo-sc.yaml
kubectl apply -f examples/kubernetes/sharepool/demo-pvc.yaml
```

*Note: Update the `sharepool` parameter in `demo-sc.yaml` with your project's share pool resource ID before applying.*

---

## Step 6: Verify the Results

Query the PVC events to verify that the CSI driver is actively routing calls to GCFS:

```bash
kubectl describe pvc filestore-share-pvc -n default
```

Look at the **Events** section at the bottom. Since this is an empty test pool, you should see the expected successful response code `429: no available shares in the pool` returned from the Filestore Autopush sandbox API:

```text
Warning  ProvisioningFailed  11s  filestore.csi.storage.gke.io  failed to provision volume with StorageClass "filestore-share-pool-sc": rpc error: code = ResourceExhausted desc = googleapi: Error 429: no available shares in the pool
```

If you see the `429` error, the E2E verification is **successful**! The driver successfully connected, authenticated, and processed the Share Pools provision API call.

---

## Cleanup

When you are finished testing, delete the test resource PVC and StorageClass to release resources:

```bash
kubectl delete -f examples/kubernetes/sharepool/demo-pvc.yaml
kubectl delete -f examples/kubernetes/sharepool/demo-sc.yaml
```
