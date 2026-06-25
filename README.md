# OCP4NV GPU Virtualization & Containerization Demo

This repository documents the deployment steps for enabling NVIDIA GPU virtualization on OpenShift using RHEL for NVIDIA (RHEL4NV) with GB200 and GH200 hardware.

## Overview

The goal of this document is to provide comprehensive steps for deploying a complete GPU virtualization stack on OpenShift, including:
- Node Feature Discovery (NFD)
- NVIDIA GPU Operator with custom drivers
- OpenShift Virtualization (CNV) with AI Enablement (AIE)
- GPU passthrough configuration for virtual machines

## Test Environment

### Single Node OpenShift (SNO) Configuration
- **Cluster Type**: Single Node OpenShift (GB configuration)
- **Node**: Dell Server 9712a
- **GPU Hardware**: 4x NVIDIA GB200 (Blackwell architecture)
- **OpenShift Version**: 4.22.0
- **Base Kernel**: 6.12.0-231.12.el10nv.aarch64_64k
- **Architecture**: ARM64

### Multi-Node Configuration (GH)
- **Cluster Type**: 3 masters + 1 worker
- **GPU Node**: NVIDIA Grace GH200
- **GPU Hardware**: 1x NVIDIA GH200 480GB
- **OpenShift Version**: 4.22.0-rc.5
- **Architecture**: ARM64

## Prerequisites

- OpenShift cluster deployed and accessible
- Cluster admin access (oc CLI authenticated)
- Custom RHCOS image for NVIDIA Grace hardware
- Access to pre-release operator catalogs (for RC versions)

## Custom Images Required

The following custom images are required for this deployment. These are not publicly available and must be built or obtained separately.

| Variable | Description | Used In |
|----------|-------------|---------|
| `OS_IMAGE_URL` | Custom RHCOS image for NVIDIA Grace hardware | Section 3a (MachineConfig osImageURL) |
| `DTK_IMAGE` | Custom Driver Toolkit image matching the osImageURL kernel | Section 6 (DTK Configuration) |
| `GPU_DRIVER_REPO` | GPU driver container image repository | Section 7b (GPU ClusterPolicy container mode) |
| `GPU_DRIVER_IMAGE` | GPU driver container image name | Section 7b (GPU ClusterPolicy container mode) |
| `GPU_DRIVER_VERSION` | GPU driver version | Section 7b (GPU ClusterPolicy container mode) |
| `AIE_LAUNCHER_IMAGE` | NVIDIA virt-launcher image for AIE (if not in CSV) | Section 9d (AIE Launcher configuration) |
| `MOFED_IMAGE` | Custom MOFED driver image name | Section 10c (NNO NicClusterPolicy) |
| `MOFED_REPO` | MOFED driver image repository | Section 10c (NNO NicClusterPolicy) |
| `MOFED_VERSION` | MOFED driver version (DOCA version) | Section 10c (NNO NicClusterPolicy) |

**Note:** The actual image references for these variables are maintained in a private repository. Contact the team for access.

## Deployment Workflow

### 1. Node Feature Discovery (NFD) Operator

NFD is required for GPU hardware discovery and node labeling.

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nfd
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nfd-operatorgroup
  namespace: openshift-nfd
spec:
  targetNamespaces:
  - openshift-nfd
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd-operator
  namespace: openshift-nfd
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for CSV to be ready
oc wait --for=condition=Succeeded csv -n openshift-nfd \
  -l operators.coreos.com/nfd.openshift-nfd --timeout=600s

# Verify NFD is running
oc get pods -n openshift-nfd
```

### 2. NVIDIA GPU Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-gpu-operatorgroup
  namespace: nvidia-gpu-operator
spec:
  targetNamespaces:
  - nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for CSV
oc wait --for=condition=Succeeded csv -n nvidia-gpu-operator \
  -l operators.coreos.com/gpu-operator-certified.nvidia-gpu-operator --timeout=600s
```

### 3. Machine Configurations

Apply MachineConfigs sequentially to avoid multiple reboots. Each MachineConfig triggers a node reboot.

**Note:** For SNO, target `master` pool. For multi-node, target `worker` pool.

#### 3a. Custom OS Image (osImageURL)

```bash
# Set your custom RHCOS image
OS_IMAGE_URL="${OS_IMAGE_URL:?OS_IMAGE_URL must be set}"

# For SNO (master pool)
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-osimage
spec:
  osImageURL: ${OS_IMAGE_URL}
EOF

# Wait for MCP to update (reboot 1)
oc wait mcp master --for condition=Updated --timeout=30m
```

For multi-node, replace `master` with `worker` in labels and name.

#### 3b. Hugepages Configuration

```bash
# For SNO (master pool)
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-hugepages
spec:
  config:
    ignition:
      version: 3.2.0
  kernelArguments:
    - hugepagesz=2M
    - hugepages=64000
EOF

# Wait for MCP to update (reboot 2)
oc wait mcp master --for condition=Updated --timeout=30m
```

### 4. CPU Manager and Kubelet Configuration

#### 4a. Label Node for CPU Manager

```bash
# For SNO
MASTER_NODE=$(oc get nodes -l node-role.kubernetes.io/master --no-headers | awk '{print $1}')
oc label node ${MASTER_NODE} cpumanager=true --overwrite=true

# For multi-node (exclude control-plane nodes that also have worker role)
WORKER_NODE=$(oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/control-plane' --no-headers | awk '{print $1}')
oc label node ${WORKER_NODE} cpumanager=true --overwrite=true
```

#### 4b. Update MachineConfigPool Label

```bash
# For SNO
oc patch machineconfigpool master --type=merge \
  -p '{"metadata":{"labels":{"custom-kubelet":"cpumanager-enabled"}}}'

# For multi-node
oc patch machineconfigpool worker --type=merge \
  -p '{"metadata":{"labels":{"custom-kubelet":"cpumanager-enabled"}}}'
```

#### 4c. Create KubeletConfig

**For SNO (4 GPUs - restricted topology):**

```bash
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpumanager-enabled
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: cpumanager-enabled
  kubeletConfig:
    cpuManagerPolicy: static
    cpuManagerReconcilePeriod: 5s
    topologyManagerPolicy: restricted
    topologyManagerPolicyOptions:
      max-allowable-numa-nodes: "34"
EOF

# Wait for MCP to update (reboot 3)
oc wait mcp master --for condition=Updated --timeout=30m
```

**For multi-node (1 GPU - single-numa-node):**

```bash
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpumanager-enabled
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: cpumanager-enabled
  kubeletConfig:
    cpuManagerPolicy: static
    cpuManagerReconcilePeriod: 5s
    topologyManagerPolicy: single-numa-node
    topologyManagerPolicyOptions:
      max-allowable-numa-nodes: "34"
EOF

# Wait for MCP to update
oc wait mcp worker --for condition=Updated --timeout=30m
```

**Note:** Total expected reboots: 3 (osImageURL, hugepages, kubelet)

### 5. Local Storage for VMs

Create local persistent volumes on the GPU node.

```bash
# For SNO
MASTER_NODE=$(oc get nodes -l node-role.kubernetes.io/master --no-headers | awk '{print $1}')
TARGET_NODE=${MASTER_NODE}

# For multi-node (exclude control-plane nodes that also have worker role)
WORKER_NODE=$(oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/control-plane' --no-headers | awk '{print $1}')
TARGET_NODE=${WORKER_NODE}

# Create directories on node (10 PVs, each VM needs 2 PVCs)
oc debug node/${TARGET_NODE} -- chroot /host bash -c "mkdir -p /mnt/local-storage/pv-{1..10}"

# Create local PVs
PV_COUNT=10
PV_SIZE="100Gi"

for i in $(seq 1 ${PV_COUNT}); do
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-local-pv-${i}
spec:
  capacity:
    storage: ${PV_SIZE}
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /mnt/local-storage/pv-${i}
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ${TARGET_NODE}
EOF
done

# Verify PVs
oc get pv | grep manual-local
```

### 6. Driver Toolkit (DTK) Configuration

When using a custom osImageURL, the `driver-toolkit` ImageStream must be tagged with a custom DTK image matching the worker node's OS version. The GPU operator uses this ImageStream to find the correct DTK for building driver modules.

```bash
# Get the target node OS version from NFD labels
# For SNO
TARGET_NODE=$(oc get nodes -l node-role.kubernetes.io/master --no-headers | awk '{print $1}')
# For multi-node
TARGET_NODE=$(oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/control-plane' --no-headers | awk '{print $1}')

OS_VERSION=$(oc get node ${TARGET_NODE} -o jsonpath='{.metadata.labels.feature\.node\.kubernetes\.io/system-os_release\.OSTREE_VERSION}')
echo "Target node OS version: ${OS_VERSION}"

# Check current DTK ImageStream tags
oc get is driver-toolkit -n openshift -o jsonpath='{range .spec.tags[*]}{.name}{": "}{.from.name}{"\n"}{end}'

# Tag custom DTK image to match the node OS version
DTK_IMAGE="${DTK_IMAGE:?DTK_IMAGE must be set}"
oc tag ${DTK_IMAGE} driver-toolkit:${OS_VERSION} -n openshift

# Verify the tag was added
oc get is driver-toolkit -n openshift -o jsonpath='{range .spec.tags[*]}{.name}{": "}{.from.name}{"\n"}{end}'
```

**Note:** The DTK image must match the kernel provided by osImageURL. Mismatch will cause GPU driver module build failures.

### 7. GPU Operator ClusterPolicy

#### 7a. VM Passthrough Mode (for CNV/Virtualization)

**Required for GPU-enabled Virtual Machines (Section 11).** The GPU operator must be in vm-passthrough mode before deploying VMs with GPU passthrough.

```bash
# Get base ClusterPolicy from CSV
CSV_NAME=$(oc get csv -n nvidia-gpu-operator -o name | grep gpu-operator-certified | sed 's|.*/||')
oc get csv ${CSV_NAME} -n nvidia-gpu-operator -o jsonpath='{.metadata.annotations.alm-examples}' | \
  jq '.[0]' > /tmp/clusterpolicy-base.json

# Modify for vm-passthrough mode
cat > /tmp/clusterpolicy-vm-passthrough.json <<'POLICY'
{
  "apiVersion": "nvidia.com/v1",
  "kind": "ClusterPolicy",
  "metadata": {
    "name": "gpu-cluster-policy"
  },
  "spec": {
    "operator": {
      "use_ocp_driver_toolkit": true
    },
    "sandboxWorkloads": {
      "enabled": true,
      "defaultWorkload": "vm-passthrough",
      "mode": "kubevirt"
    },
    "sandboxDevicePlugin": {
      "enabled": true
    },
    "vfioManager": {
      "enabled": true
    },
    "driver": {
      "enabled": true
    },
    "toolkit": {
      "enabled": true
    },
    "devicePlugin": {
      "enabled": true
    },
    "dcgm": {
      "enabled": true
    },
    "dcgmExporter": {
      "enabled": true
    },
    "daemonsets": {
      "updateStrategy": "RollingUpdate"
    },
    "gfd": {
      "enabled": true
    },
    "migManager": {
      "enabled": true
    },
    "nodeStatusExporter": {
      "enabled": true
    },
    "mig": {
      "strategy": "single"
    }
  }
}
POLICY

cat /tmp/clusterpolicy-vm-passthrough.json | oc apply -f -

# Wait for vfio-manager to be ready
sleep 30
oc get pods -n nvidia-gpu-operator -l app=nvidia-vfio-manager
```

#### 7b. Container Mode (for container workloads)

```bash
GPU_DRIVER_IMAGE="<your-driver-image-name>"
GPU_DRIVER_REPO="<your-driver-registry>"
GPU_DRIVER_VERSION="<driver-version>"

cat > /tmp/clusterpolicy-container.json <<POLICY
{
  "apiVersion": "nvidia.com/v1",
  "kind": "ClusterPolicy",
  "metadata": {
    "name": "gpu-cluster-policy"
  },
  "spec": {
    "operator": {
      "use_ocp_driver_toolkit": true
    },
    "sandboxWorkloads": {
      "enabled": false,
      "defaultWorkload": "container",
      "mode": "kubevirt"
    },
    "driver": {
      "enabled": true,
      "repository": "${GPU_DRIVER_REPO}",
      "version": "${GPU_DRIVER_VERSION}",
      "image": "${GPU_DRIVER_IMAGE}"
    },
    "toolkit": {
      "enabled": true
    },
    "devicePlugin": {
      "enabled": true
    },
    "dcgm": {
      "enabled": true
    },
    "dcgmExporter": {
      "enabled": true
    },
    "daemonsets": {
      "updateStrategy": "RollingUpdate"
    },
    "gfd": {
      "enabled": true
    },
    "migManager": {
      "enabled": true
    },
    "nodeStatusExporter": {
      "enabled": true
    },
    "vfioManager": {
      "enabled": false
    }
  }
}
POLICY

cat /tmp/clusterpolicy-container.json | oc apply -f -
```

**Important:** When switching from vm-passthrough to container mode, stop all VMs using GPU passthrough first. The vfio-pci driver cannot be unbound while VMs have GPU devices attached.

### 8. Verify GPU Resources

```bash
# For SNO
NODE_NAME=$(oc get nodes -l node-role.kubernetes.io/master --no-headers | awk '{print $1}')

# For multi-node (exclude control-plane nodes that also have worker role)
NODE_NAME=$(oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/control-plane' --no-headers | awk '{print $1}')

# Check allocatable resources
oc describe node ${NODE_NAME} | grep -A15 "Allocatable:"

# Expected in vm-passthrough mode:
# - nvidia.com/GB100_HGX_GB200: 4 (or nvidia.com/GH200_120GB: 1)
# - devices.kubevirt.io/iommufd: 110
# - hugepages-2Mi: 125Gi

# Expected in container mode:
# - nvidia.com/gpu: 4 (or 1 for single GPU)
# - devices.kubevirt.io/iommufd: 0

# Verify vfio-pci driver binding (vm-passthrough mode)
VFIO_POD=$(oc get pods -n nvidia-gpu-operator -l app=nvidia-vfio-manager -o name | head -1)
oc logs ${VFIO_POD} -n nvidia-gpu-operator | grep nvgrace_gpu_vfio_pci

# Verify NVIDIA driver (container mode)
oc debug node/${NODE_NAME} -- chroot /host lsmod | grep nvidia
```

### 9. OpenShift Virtualization with AIE

Deploy OpenShift Virtualization (CNV) with AI Enablement (AIE) for GPU passthrough with vIOMMU support.

#### 9a. Install CNV Operator

```bash
# Check if CNV is available in current catalog
oc get packagemanifests | grep kubevirt-hyperconverged

# Install from official catalog
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
  - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  channel: stable
EOF

# Wait for CSV to succeed
sleep 60
INSTALLED_CSV=$(oc get csv -n openshift-cnv -o name | grep kubevirt-hyperconverged-operator | sed 's|.*/||')
oc wait csv ${INSTALLED_CSV} -n openshift-cnv \
  --for jsonpath='{.status.phase}'=Succeeded --timeout=10m
```

#### 9b. Deploy HyperConverged CR

```bash
cat <<EOF | oc apply -f -
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec: {}
EOF

# Wait for HyperConverged to be available
oc wait HyperConverged kubevirt-hyperconverged -n openshift-cnv \
  --for condition=Available --timeout=30m
```

#### 9c. Enable AIE (AI Enablement)

```bash
# Annotate HyperConverged to enable AIE
oc annotate HyperConverged kubevirt-hyperconverged -n openshift-cnv \
  hco.kubevirt.io/deployAIE=true

# Verify AIE resources
sleep 15
oc get deployment kubevirt-aie-webhook -n openshift-cnv
oc get daemonset iommufd-device-plugin -n openshift-cnv
oc get configmap kubevirt-aie-launcher-config -n openshift-cnv
```

#### 9d. Configure AIE Launcher for NVIDIA GPUs

```bash
INSTALLED_CSV=$(oc get csv -n openshift-cnv -o name | grep kubevirt-hyperconverged-operator | sed 's|.*/||')

# Try to extract NVIDIA launcher from CSV relatedImages
LAUNCHER_NV_IMAGE=$(oc get csv ${INSTALLED_CSV} -n openshift-cnv -o json \
  | jq -r '.spec.relatedImages[] | select(.name | test("virt-launcher-4-nv")) | .image')

# If not in CSV, set manually (required for some builds)
if [ -z "${LAUNCHER_NV_IMAGE}" ]; then
  echo "NVIDIA launcher not found in CSV - provide AIE_LAUNCHER_IMAGE manually"
  # Example: AIE_LAUNCHER_IMAGE="quay.io/openshift-cnv/container-native-virtualization-virt-launcher-4-nv-rhel10:<tag>"
  LAUNCHER_NV_IMAGE=${AIE_LAUNCHER_IMAGE}
fi

echo "NVIDIA virt-launcher image: ${LAUNCHER_NV_IMAGE}"

# Configure AIE launcher for NVIDIA GPUs
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-aie-launcher-config
  namespace: openshift-cnv
data:
  config.yaml: |
    rules:
    - name: "nvidia-gpu-launcher"
      image: "${LAUNCHER_NV_IMAGE}"
      selector:
        deviceNames:
        - "nvidia.com/GH200_120GB"
        - "nvidia.com/GH100"
        - "nvidia.com/GH100_GH200_120GB___480GB"
        - "nvidia.com/GB100_HGX_GB200"
    - name: "labeled-vms"
      image: "${LAUNCHER_NV_IMAGE}"
      selector:
        vmLabels:
          matchLabels:
            aie.kubevirt.io/launcher: "true"
EOF
```

#### Backup: Deploy from CNV Nightly Catalog

Use this when CNV is not yet available in the official `redhat-operators` catalog (e.g., pre-release OCP versions).

**Note:** Nightly builds may not include the NVIDIA virt-launcher image. In that case, provide `AIE_LAUNCHER_IMAGE` manually.

```bash
CNV_VERSION="4.22"

# Create nightly catalog
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cnv-nightly-catalog-source
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/openshift-cnv/nightly-catalog:${CNV_VERSION}
  displayName: OpenShift Virtualization Nightly Index
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 8h
EOF

oc wait --for=condition=Ready pod -l olm.catalogSource=cnv-nightly-catalog-source \
  -n openshift-marketplace --timeout=300s

# Extract starting CSV and install
STARTING_CSV=$(oc get packagemanifest -l catalog=cnv-nightly-catalog-source \
  -o jsonpath="{$.items[?(@.metadata.name=='kubevirt-hyperconverged')].status.channels[?(@.name==\"nightly-${CNV_VERSION}\")].currentCSV}")

# Use source: cnv-nightly-catalog-source and channel: nightly-${CNV_VERSION} in the Subscription
# Then continue with steps 9b onwards
```

### 10. NVIDIA Network Operator (NNO)

Deploy NNO for Mellanox ConnectX NIC OFED drivers.

**Prerequisites:** NFD must be configured with PCI class `02` (Ethernet) in deviceClassWhitelist to discover Mellanox NICs.

#### 10a. Configure NFD for Mellanox Discovery

```bash
# Update NFD config to include Ethernet devices (class 02)
oc patch nodefeaturediscovery nfd-instance -n openshift-nfd --type=merge -p '
spec:
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist: ["02", "03", "0b40", "12"]
          deviceLabelFields: ["class", "vendor"]
'

# Restart NFD worker on GPU node to pick up config
TARGET_NODE="<gpu-node-name>"
NFD_WORKER=$(oc get pods -n openshift-nfd --field-selector spec.nodeName=${TARGET_NODE} -o name | grep worker | head -1)
oc delete ${NFD_WORKER} -n openshift-nfd
sleep 30

# Verify Mellanox labels
oc get nodes ${TARGET_NODE} -o json | jq '.metadata.labels | to_entries[] | select(.key | test("15b3")) | "\(.key)=\(.value)"'

# NNO expects pci-15b3.present (without class prefix) - add manually
oc label node ${TARGET_NODE} feature.node.kubernetes.io/pci-15b3.present=true --overwrite
```

#### 10b. Install NNO Operator

NNO may not be available in the current OCP catalog. Use the previous version catalog if needed.

```bash
OCP_PREVIOUS_VERSION="4.21"

# Check availability
oc get packagemanifests | grep nvidia-network-operator

# If not found, create previous version catalog
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: certified-operators-${OCP_PREVIOUS_VERSION//./-}
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.redhat.io/redhat/certified-operator-index:v${OCP_PREVIOUS_VERSION}
  displayName: Certified Operators (${OCP_PREVIOUS_VERSION})
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 30m
EOF

oc wait --for=condition=Ready pod -l olm.catalogSource=certified-operators-${OCP_PREVIOUS_VERSION//./-} \
  -n openshift-marketplace --timeout=300s

# Install NNO
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-network-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-network-operatorgroup
  namespace: nvidia-network-operator
spec:
  targetNamespaces:
  - nvidia-network-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nvidia-network-operator
  namespace: nvidia-network-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nvidia-network-operator
  source: certified-operators-${OCP_PREVIOUS_VERSION//./-}
  sourceNamespace: openshift-marketplace
EOF

sleep 60
INSTALLED_CSV=$(oc get csv -n nvidia-network-operator -o name | grep nvidia-network-operator | sed 's|.*/||')
oc wait csv ${INSTALLED_CSV} -n nvidia-network-operator \
  --for jsonpath='{.status.phase}'=Succeeded --timeout=10m
```

#### 10c. Create NicClusterPolicy

```bash
MOFED_IMAGE="<your-mofed-driver-image>"
MOFED_REPO="<your-registry>"
MOFED_VERSION="<doca-version>"

cat <<EOF | oc apply -f -
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  nicFeatureDiscovery:
    image: nic-feature-discovery
    repository: ghcr.io/mellanox
    version: v0.0.1
  secondaryNetwork:
    ipoib:
      image: ipoib-cni
      repository: ghcr.io/mellanox
      version: v1.2.0
  nvIpam:
    enableWebhook: false
    image: nvidia-k8s-ipam
    repository: ghcr.io/mellanox
    version: v0.2.0
  ofedDriver:
    forcePrecompiled: false
    image: ${MOFED_IMAGE}
    imagePullSecrets: []
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
    repository: ${MOFED_REPO}
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 20
    terminationGracePeriodSeconds: 300
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: true
        enable: true
        force: true
        podSelector: ""
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      safeLoad: false
    version: ${MOFED_VERSION}
    env:
    - name: UNLOAD_STORAGE_MODULES
      value: "true"
    - name: RESTORE_DRIVER_ON_POD_TERMINATION
      value: "true"
    - name: CREATE_IFNAMES_UDEV
      value: "true"
    - name: ENTRYPOINT_DEBUG
      value: "true"
  rdmaSharedDevicePlugin:
    config: |
      {
        "configList": [
          {
            "resourceName": "rdma_shared_device_a",
            "rdmaHcaMax": 63,
            "selectors": {
              "vendors": ["15b3"]
            }
          }
        ]
      }
    image: k8s-rdma-shared-dev-plugin
    imagePullSecrets: []
    repository: nvcr.io/nvidia/mellanox
    version: sha256:c50fe96cfb2e32cf497f3723cc0b90f72582c2bc84b3593389dc14bc813a58fc
EOF
```

**Note:** The NNO constructs the full MOFED image tag as `<repo>/<image>:<version>-<os_id><version_id>-<arch>` (e.g., `quay.io/user/my-driver:doca3.4.0-26.04-0.8.6.0-0-rhel10.2-arm64`). The image must exist in the registry with this exact tag.

#### 10d. Verify NNO

```bash
# Wait for MOFED pod (may take 5-10 minutes)
oc get pods -n nvidia-network-operator | grep mofed

# Verify mlx5 modules loaded
MOFED_POD=$(oc get pods -n nvidia-network-operator --no-headers | grep mofed | awk '{print $1}')
oc exec -n nvidia-network-operator ${MOFED_POD} -c mofed-container -- lsmod | grep mlx5

# Verify OFED installation
oc exec -n nvidia-network-operator ${MOFED_POD} -c mofed-container -- ofed_info | tail -20

# Check NicClusterPolicy status
oc get nicclusterpolicy nic-cluster-policy -o jsonpath='{.status.state}'
```

### 11. Deploy GPU-Enabled Virtual Machine

**Prerequisites:**
- GPU Operator must be configured in **vm-passthrough mode** (Section 7a)
- CNV with AIE must be deployed and available (Section 9)

#### 11a. Create VM and install driver
Create a VM that has installed the nvidia-gpu-driver for Grace Hopper/Blackwell. ( eg: 590.48.01 )

#### 11b. Configure VM with GPU Passthrough

```bash
# Extract GPU device name
NODE_NAME=$(oc get nodes -l node-role.kubernetes.io/master --no-headers | awk '{print $1}')  # SNO
# NODE_NAME=$(oc get nodes -l node-role.kubernetes.io/worker --no-headers | awk '{print $1}')  # Multi-node

GPU_DEVICE_NAME=$(oc describe node ${NODE_NAME} | grep "nvidia.com/GB" | awk '{print $1}')
echo "GPU Device: ${GPU_DEVICE_NAME}"

# Patch VM with GPU configuration
cat > /tmp/vm-gpu-patch.yaml <<EOF
spec:
  template:
    spec:
      domain:
        devices:
          gpus:
          - deviceName: ${GPU_DEVICE_NAME}
            name: gpu1
        cpu:
          cores: 2
          dedicatedCpuPlacement: true
          numa:
            guestMappingPassthrough: {}
        memory:
          guest: 16Gi
          hugepages:
            pageSize: 2Mi
        resources:
          limits:
            devices.kubevirt.io/iommufd: "1"
EOF

oc patch vm gpu-test-vm -n gpu-test-vm --type=merge --patch-file=/tmp/vm-gpu-patch.yaml

# Start VM
oc patch vm gpu-test-vm -n gpu-test-vm --type=merge -p '{"spec":{"running":true}}'

# Wait for VM to be ready
oc wait vm gpu-test-vm -n gpu-test-vm --for condition=Ready --timeout=10m
```

#### 11c. Verify GPU in VM


```bash
# SSH into VM (user: <username>, password: <password>)
virtctl ssh --namespace gpu-test-vm --username=<username> vm/gpu-test-vm

# Inside VM: Check GPU
lspci -nn | grep -i nvi

# Expected output for GB200:
# NVIDIA Corporation GB200
```

#### 11d. Build and run CUDA tests
Clone the https://github.com/nvidia/cuda-samples build and run the tests

```
sudo su
cd cuda-samples/
python3 ./run_tests.py --dir ./build/Samples/ --output res --parallel 2
```

#### 11e. Verify AIE Configuration

```bash
# Check virt-launcher is using NVIDIA image
oc get pods -n gpu-test-vm -l kubevirt.io/domain=gpu-test-vm -o yaml | grep image:

# Expected: virt-launcher-4-nv-rhel10

# Check iommufd resource injection
oc get pods -n gpu-test-vm -l kubevirt.io/domain=gpu-test-vm -o yaml | grep iommufd

# Expected: devices.kubevirt.io/iommufd: "1"
```

## Validation Steps

### Node Resources
```bash
# Check node allocatable (vm-passthrough mode)
oc describe node <node-name> | grep -A15 "Allocatable:"
# Should show:
# - nvidia.com/GB100_HGX_GB200: 4 (or appropriate device)
# - devices.kubevirt.io/iommufd: 110
# - hugepages-2Mi: 125Gi
```

### GPU Operator Pods
```bash
# All pods should be Running or Completed
oc get pods -n nvidia-gpu-operator

# Expected pods (vm-passthrough):
# - nvidia-vfio-manager: Running
# - nvidia-sandbox-device-plugin: Running
# - nvidia-sandbox-validator: Running

# Expected pods (container mode):
# - nvidia-driver-daemonset: Running
# - nvidia-device-plugin: Running
# - nvidia-container-toolkit: Running
# - nvidia-cuda-validator: Completed
```

### CNV Status
```bash
oc get hyperconverged -n openshift-cnv
oc get pods -n openshift-cnv | grep -E "aie|iommufd"
```

### VM Status
```bash
oc get vm -A
oc get vmi -A
```

## Troubleshooting

### Mode Switching

When switching GPU operator from vm-passthrough to container mode:
1. **Stop all VMs first** that are using GPU passthrough
2. Delete the ClusterPolicy
3. Create new ClusterPolicy with desired mode
4. Wait for operator to reconfigure

The vfio-pci driver cannot be unbound while VMs have GPU devices attached.

### Driver Logs
```bash
# vfio-manager logs (vm-passthrough)
oc logs -n nvidia-gpu-operator -l app=nvidia-vfio-manager

# Driver daemonset logs (container mode)
oc logs -n nvidia-gpu-operator -l app=nvidia-driver-daemonset
```

### MachineConfig Issues
```bash
# Check MCP status
oc get mcp

# Check node status
oc get nodes

# Debug node
oc debug node/<node-name> -- chroot /host systemctl status machine-config-daemon
```

## References

- [OpenShift Virtualization Documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
- [Node Feature Discovery](https://kubernetes-sigs.github.io/node-feature-discovery/stable/get-started/index.html)
