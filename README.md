# hcloud-autoscaler

Helm chart for deploying a full Cluster Autoscaler stack on Hetzner Cloud with Rancher/RKE2 support.

> **[Читать на русском →](README.ru.md)**

## What's Included

| Component | Description |
|---|---|
| **cluster-autoscaler** | Deployment with Hetzner Cloud provider, RBAC, Secret with hcloud config |
| **node-finalizer-cleaner** | CronJob (every 2 min) removes Rancher `wrangler.cattle.io/*` finalizers from nodes marked for deletion by autoscaler |
| **protected-nodes hook** | Post-install Job: annotates permanent nodes with `scale-down-disabled=true` |

## The Problem This Solves

When cluster-autoscaler deletes an hcloud VM, the node becomes `unreachable`.
Rancher adds finalizers `wrangler.cattle.io/node` and `wrangler.cattle.io/managed-etcd-controller`
to all nodes, blocking deletion of the Kubernetes node object.
The result: nodes hang for hours in `NotReady,SchedulingDisabled` state.

`node-finalizer-cleaner` waits `graceSeconds` (default: 300s), then:
1. Removes finalizers
2. Deletes the node object
3. *(Optional)* Deletes the corresponding machine record from Rancher UI via API

**Trigger conditions** (both must be true):
- taint `ToBeDeletedByClusterAutoscaler` — autoscaler marked the node
- taint `node.kubernetes.io/unreachable` — node is unreachable

## Installation

```bash
# Copy and fill in real values (never commit this file!)
cp values.yaml values.local.yaml
# Fill in: hcloud.token, hcloud.sshKey, hcloud.network, cloudInitBase64

helm upgrade --install hcloud-autoscaler . \
  -n cluster-autoscaler \
  --create-namespace \
  -f values.yaml \
  -f values.local.yaml
```

## Node Groups — How It Works

Node group configuration is set in **two places** in values.yaml:

### 1. `autoscaler.nodeGroups` — limits and instance type

Tells the autoscaler which groups to track and their scaling boundaries:

```yaml
autoscaler:
  nodeGroups:
    - name: cx23-workers    # group name (must match hcloud.nodeGroups key)
      instanceType: cx23    # Hetzner instance type (cx22, cx32, cx52, etc.)
      region: fsn1          # region (fsn1, nbg1, hel1, ash, sin)
      min: 1                # minimum nodes (0 = group can be empty)
      max: 5                # maximum nodes
```

### 2. `hcloud.nodeGroups` — what to create in Hetzner

Config for the Hetzner API: which image, SSH key, network, and most importantly — the cloud-init script:

```yaml
hcloud:
  token: ""
  image: ubuntu-24.04
  sshKey: ""        # SSH key name in Hetzner
  network: ""       # private network name
  nodeGroups:
    cx23-workers:
      cloudInitBase64: ""   # base64-encoded cloud-init for Rancher registration
    cx33-workers:
      cloudInitBase64: ""
```

Generate `cloudInitBase64`:
```bash
base64 -w0 examples/cloud-init-cx23.yaml
```

### Scaling Logic

**Scale-up:**
1. Autoscaler detects Pending pods
2. Simulates which group can fit them
3. Picks the group with lowest waste (CPU/memory)
4. Increases size via Hetzner API → VM is created → cloud-init registers node in Rancher
5. Pods are scheduled on the new node

**Scale-down:**
1. Node is idle longer than `scaleDownUnneededTime` (default: 5m)
2. Cooldown after last scale-up has passed (`scaleDownDelayAfterAdd`: 5m)
3. Autoscaler taints node with `ToBeDeletedByClusterAutoscaler`, drains it, deletes VM in Hetzner
4. Node becomes `unreachable` → Rancher adds wrangler finalizers
5. `node-finalizer-cleaner` waits 300s, removes finalizers, deletes the node object, and removes the machine from Rancher UI

**Protected nodes** (`protectedNodes.nodes`) receive annotation
`cluster-autoscaler.kubernetes.io/scale-down-disabled=true` via post-install hook — autoscaler will never scale them down.

### Adding a New Node Group

1. Create a cloud-init script based on `examples/cloud-init-cx23.yaml` (update `instanceType` in labels)
2. Add to `autoscaler.nodeGroups`:
   ```yaml
   - name: cx42-workers
     instanceType: cx42
     region: fsn1
     min: 0
     max: 2
   ```
3. Add to `hcloud.nodeGroups`:
   ```yaml
   cx42-workers:
     cloudInitBase64: "<base64 of new cloud-init>"
   ```
4. `helm upgrade hcloud-autoscaler . -f values.yaml -f values.local.yaml`

## Rancher Integration (optional)

When `rancher.enabled: true`, after deleting a k8s node the cleaner also calls the Rancher API
to remove the `Nodenotfound` / `Failed` machine record from the Rancher UI.

```yaml
# values.local.yaml
rancher:
  enabled: true
  url: "https://your-rancher.example.com"
  token: "token-xxxxx:yyyyyyy"   # Rancher → Account & API Keys → Create API Key
  clusterId: "c-m-xxxxxxxx"      # visible in Rancher URL when viewing the cluster
```

## cloud-init

The cloud-init for nodes must:
1. Set up default route via `10.102.24.1` (hcloud private network gateway)
2. Configure DNS
3. Fetch hcloud server ID from metadata and write to RKE2 config as `provider-id=hcloud://<id>`
4. Add node labels: `hcloud/node-group=<name>`, `node.kubernetes.io/instance-type=<type>`
5. Run Rancher system-agent to register in the cluster

See example: `examples/cloud-init-cx23.yaml`

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `autoscaler.nodeGroups` | cx23(1-5), cx33(0-3) | Node group configuration |
| `autoscaler.scaleDownUnneededTime` | `5m` | How long before removing idle nodes |
| `autoscaler.scaleDownDelayAfterAdd` | `5m` | Scale-down cooldown after scale-up |
| `finalizerCleaner.enabled` | `true` | Enable cleaner |
| `finalizerCleaner.graceSeconds` | `300` | Grace period before cleaning finalizers |
| `finalizerCleaner.schedule` | `*/2 * * * *` | CronJob schedule |
| `rancher.enabled` | `false` | Enable Rancher machine cleanup |
| `rancher.url` | `""` | Rancher URL |
| `rancher.clusterId` | `""` | Rancher cluster ID (e.g. `c-m-5g86mm7f`) |
| `protectedNodes.nodes` | masters + workers | Nodes protected from scale-down |

## Requirements

- Kubernetes 1.28+
- Rancher/RKE2
- hcloud private network for nodes (e.g. 10.102.24.0/24)
- hcloud API token with Read/Write access to Servers, Networks, SSHKeys

## Repository Structure

```
hcloud-autoscaler/
├── Chart.yaml
├── values.yaml              # Public values (no secrets)
├── values.local.yaml        # Real credentials — add to .gitignore!
├── .gitignore
├── templates/
│   ├── namespace.yaml
│   ├── autoscaler-secret.yaml
│   ├── autoscaler-rbac.yaml
│   ├── autoscaler-deployment.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── configmap.yaml
│   ├── rancher-secret.yaml
│   ├── finalizer-cleaner-cronjob.yaml
│   └── protected-nodes-hook.yaml
└── examples/
    └── cloud-init-cx23.yaml
```

---

## Professional Support

Need help deploying this chart, setting up Kubernetes infrastructure, or Hetzner/Rancher integrations?

We provide professional DevOps and infrastructure engineering services:

- 🌐 **[nexops.eu](https://nexops.eu/en/)** — DevOps & Cloud engineering (EN)
- 🌐 **[it-24.pro](https://it-24.pro/)** — IT infrastructure & managed services (RU/UA)

Feel free to open an issue or reach out directly.
