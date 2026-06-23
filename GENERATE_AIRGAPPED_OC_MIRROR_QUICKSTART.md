# Airgapped OC-Mirror — Portworx Quickstart

End-to-end quickstart for mirroring **OpenShift + operators + Portworx images** into a disconnected registry. Pairs the baseline oc-mirror flow from [OC_MIRROR.md](./OC_MIRROR.md) with the Portworx-specific playbook [generate-airgapped-oc-mirror.yaml](./generate-airgapped-oc-mirror.yaml).

## What this gives you

1. A mirror host with `oc`, `oc-mirror`, and `mirror-registry` installed.
2. An `imageset-config.yaml` describing OpenShift platform release(s), operator catalogs, and additional images to mirror.
3. A generated **px-versions ConfigMap** (`px-versions-configmap.yaml`) listing the exact Portworx component images for a given `px_version` / `kbver` / `opver`.
4. A generated **Portworx ImageSetConfiguration** (`sample.yaml`) containing those Portworx images as `additionalImages`, ready to merge into the baseline imageset.
5. A single `oc-mirror` bundle that contains everything for the disconnected install.

---

## Prerequisites

- RHEL 9 mirror host with outbound internet to `mirror.openshift.com`, `registry.redhat.io`, `install.portworx.com`, and `docker.io`.
- Red Hat pull secret (from console.redhat.com).
- Disk: budget ~300–500 GB for a full OpenShift + operators + Portworx bundle.
- Python + Ansible installed (`pip install ansible` is enough — the playbook only uses `ansible.builtin`).

---

## Step 1 — Install oc-mirror tooling

Pick a desired OpenShift version (example uses `4.20.20`):

```bash
OCP_VERSION=4.20.20
BASE=https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}

curl -LO ${BASE}/oc-mirror.rhel9.tar.gz
curl -LO ${BASE}/openshift-client-linux-amd64-rhel9-${OCP_VERSION}.tar.gz
curl -LO https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz

tar -xvf openshift-client-linux-amd64-rhel9-${OCP_VERSION}.tar.gz
tar -xvf oc-mirror.rhel9.tar.gz
tar -xvf mirror-registry-amd64.tar.gz

sudo mv oc kubectl oc-mirror /usr/local/bin
sudo chmod +x /usr/local/bin/oc-mirror

oc version
oc-mirror version
```

## Step 2 — Pull secret + registry trust

```bash
mkdir -p ~/.docker
cat pullSecret >> ~/.docker/config.json

sudo mkdir -p /etc/containers/registries.d/
sudo tee /etc/containers/registries.d/registry.connect.redhat.com.yaml >/dev/null <<'EOF'
docker:
  registry.connect.redhat.com:
    use-sigstore-attachments: false
EOF
```

The second file lets oc-mirror skip signature verification on the certified-operator catalog (which hosts `portworx-certified`).

---

## Step 3 — Generate the Portworx pieces (this playbook)

The playbook [generate-airgapped-oc-mirror.yaml](./generate-airgapped-oc-mirror.yaml) does two things:

- Fetches `https://install.portworx.com/<px_version>/version?kbver=<kbver>&opver=<opver>` to discover the exact component image set Portworx will use.
- Renders two files:
  - **`px-versions-configmap.yaml`** — the `px-versions` ConfigMap to apply in-cluster after install so the Portworx Operator pulls from the mirror with pinned images.
  - **`sample.yaml`** — a partial `ImageSetConfiguration` containing only Portworx `additionalImages` (host-qualified, `docker.io/...` prepended where needed).

### Inputs

Copy the sample vars and edit:

```bash
cp sample_inputs/generate-airgapped-oc-mirror.yaml group_vars/px-airgap.yaml
$EDITOR group_vars/px-airgap.yaml
```

Key vars (full list in [sample_inputs/generate-airgapped-oc-mirror.yaml](./sample_inputs/generate-airgapped-oc-mirror.yaml)):

| Var                    | Default                                            | Meaning |
|------------------------|----------------------------------------------------|---------|
| `px_version`           | `3.6.1`                                            | Portworx release (URL path segment). |
| `kbver`                | `1.32.7`                                           | Target Kubernetes version. |
| `opver`                | `26.2.0`                                           | Portworx Operator version. |
| `namespace`            | `portworx`                                         | Namespace stamped into the ConfigMap. |
| `configmap_name`       | `px-versions`                                      | ConfigMap `metadata.name`. |
| `configmap_out`        | `{{ playbook_dir }}/px-versions-configmap.yaml`    | Output path for the ConfigMap. |
| `imageset_out`         | `{{ playbook_dir }}/../sample.yaml`                | Output path for the Portworx ImageSetConfiguration fragment. |
| `imageset_api_version` | `mirror.openshift.io/v2alpha1`                     | oc-mirror v2 API. |
| `categories`           | `[]`                                               | Extra component categories to include. `default` is always on. |
| `component_categories` | (see playbook)                                     | Category → component-key mapping. Override only to reshape groupings. |

### Categories

`default` is always selected (stork, csi-*, kube-*, pause, pxLibUpdate, clusterDiags). Opt in to any subset of:

- `autopilot`
- `telemetry`
- `monitoring` (prometheus + alertmanager — only needed if you let Portworx own monitoring)
- `px-cache`
- `fusion`
- `ocp-console-plugin`
- `cert-manager`

Example: monitoring + cert-manager:

```yaml
categories:
  - monitoring
  - cert-manager
```

Unknown categories fail fast. Components present in the upstream manifest but in no category are logged as a warning so you can decide whether to add them.

### Run it

```bash
ansible-playbook generate-airgapped-oc-mirror.yaml -e @group_vars/px-airgap.yaml
```

Or inline override:

```bash
ansible-playbook generate-airgapped-oc-mirror.yaml \
  -e px_version=3.6.1 -e kbver=1.32.7 -e opver=26.2.0 \
  -e '{"categories":["monitoring","cert-manager"]}'
```

Outputs:

```
Categories selected: ['default', 'monitoring', 'cert-manager']
Wrote ConfigMap: .../px-versions-configmap.yaml
Wrote ImageSetConfiguration: .../sample.yaml
Image count: 27
```

---

## Step 4 — Merge into the master ImageSetConfiguration

The Portworx fragment from Step 3 (`sample.yaml`) contains only `mirror.additionalImages`. Splice those entries into the baseline `imageset-config.yaml` from [OC_MIRROR.md](./OC_MIRROR.md) so a **single** oc-mirror run pulls platform + operators + Portworx:

```yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
      - name: stable-4.20
        minVersion: 4.20.20
        maxVersion: 4.20.23
    graph: true
  additionalImages:
    - name: registry.redhat.io/ubi8/ubi:latest
    - name: registry.redhat.io/ubi9/ubi:latest
    # --- BEGIN: paste mirror.additionalImages from generated sample.yaml ---
    - name: docker.io/portworx/oci-monitor:3.6.1
    - name: docker.io/openstorage/stork:25.4.0
    # ...etc, full list from sample.yaml...
    # --- END ---
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
      packages:
        - name: kubernetes-nmstate-operator
        - name: local-storage-operator
        # ...etc, see OC_MIRROR.md...
    - catalog: registry.redhat.io/redhat/certified-operator-index:v4.20
      packages:
        - name: portworx-certified
```

> `portworx-certified` in the operators block ships the **Operator**. The `additionalImages` block ships the **runtime component images** the Operator references at install time. You need both.

To do the merge programmatically:

```bash
yq eval-all '. as $item ireduce ({}; . *+ $item)' imageset-config.yaml sample.yaml > merged-imageset.yaml
```

## Step 5 — Pull the bundle

```bash
oc-mirror --v2 \
  -c merged-imageset.yaml \
  --cache-dir=/path-to-large-drive/bundle-cache \
  file://./mirror-${OCP_VERSION} \
  --retry-times 60 --image-timeout 60m --parallel-images 4 --retry-delay 5s
```

Move the resulting `mirror-${OCP_VERSION}` directory to the disconnected side and run the matching `oc-mirror ... docker://<your-registry>` push step there.

---

## Step 6 — Apply px-versions in the disconnected cluster

After Portworx Operator is installed and before (or while) you create the StorageCluster, apply the generated ConfigMap so the Operator resolves component images from your mirror at the pinned versions:

```bash
oc apply -f px-versions-configmap.yaml
```

Then in the StorageCluster spec, point at the ConfigMap:

```yaml
spec:
  image: <mirror-host>/portworx/oci-monitor:3.6.1
  customImageRegistry: <mirror-host>
  # versions configmap is auto-detected when named px-versions in the same namespace
```

---

## Troubleshooting

- **404 on `install.portworx.com`** — verify `px_version` path segment exists; try a tagged release (e.g. `3.6.1`, not `latest`).
- **"Unknown category 'X'"** — typo in `categories`; valid keys come from `component_categories` in the playbook.
- **Component in manifest but not mirrored** — appears in the "Uncategorized components" debug line. Either add it to a category locally via `component_categories` override, or extend the playbook.
- **`additionalImages` entry missing registry host** — the playbook prepends `docker.io/` only when there is no host. Upstream entries that already include `registry.connect.redhat.com/...` are left alone.
- **oc-mirror signature failures on certified catalog** — confirm `/etc/containers/registries.d/registry.connect.redhat.com.yaml` exists (Step 2).
