# HCP cluster with TPM2 disk encryption

Manifests for a Hosted Control Planes (HyperShift) cluster on the Agent platform (bare metal).
The worker NodePool uses MachineConfigs to:

- encrypt the RHCOS root disk with LUKS, unlocked automatically by the host TPM 2.0 (Clevis)
- set a password for the `core` user (console login)

Target: OpenShift 4.20, applied on the hub (MCE / ACM) cluster.

## Layout

| File | What it does |
|---|---|
| `manifests/00-namespaces.yaml` | `clusters` (HostedCluster, NodePool, config ConfigMaps) and `hcp-luks-agents` (InfraEnv, Agents). The control plane namespace `clusters-hcp-luks` is created by HyperShift. |
| `manifests/01-secrets.yaml` | Pull secret (in both namespaces) and SSH public key. |
| `manifests/02-infraenv.yaml` | Discovery ISO for the workers. Labels every Agent `hcp-luks.example.com/tpm2: "true"`. |
| `manifests/10-hostedcluster.yaml` | Hosted control plane: release, networking, managed etcd, service publishing (API via LoadBalancer, others via Route). |
| `manifests/20-nodepool-config-luks-tpm2.yaml` | ConfigMap with MachineConfig `99-worker-luks-tpm2`: LUKS on `/dev/disk/by-partlabel/root`, Clevis `tpm2: true`, root re-created as XFS on `/dev/mapper/root`. |
| `manifests/21-nodepool-config-core-password.yaml` | ConfigMap with MachineConfig `99-worker-core-password`: sets `passwordHash` for user `core`. |
| `manifests/22-nodepool.yaml` | Worker NodePool: selects TPM-labelled Agents, references both ConfigMaps in `spec.config`, upgrade type `Replace`. |
| `manifests/kustomization.yaml` | Applies everything in order. |
| `butane/*.bu` | Butane sources of the two MachineConfigs. Regenerate with `butane <file>.bu`. |

## How it fits together

1. Hosts boot the InfraEnv discovery ISO and register as Agents with the TPM label.
2. The NodePool claims approved Agents matching `agentLabelSelector`.
3. The HyperShift ignition server renders the NodePool `spec.config` MachineConfigs into the worker ignition.
4. On first boot, Ignition encrypts the root partition and binds the key to the TPM. Following boots unlock it without a passphrase.

## Prerequisites

- MCE / ACM hub with the HyperShift and Agent (assisted installer) components enabled.
- MetalLB (or another LoadBalancer provider) on the hub for the API service.
- Worker hosts with TPM 2.0 enabled in firmware.

## Usage

Fill the placeholders:

| Placeholder | File(s) |
|---|---|
| `<PULL_SECRET_JSON>` | `01-secrets.yaml` |
| `<SSH_PUBLIC_KEY>` | `01-secrets.yaml`, `02-infraenv.yaml` |
| `<OCP_VERSION>` | `10-hostedcluster.yaml`, `22-nodepool.yaml` |
| `<BASE_DOMAIN>`, `<WORKER_MACHINE_CIDR>`, `<ETCD_STORAGE_CLASS>` | `10-hostedcluster.yaml` |
| `<CORE_PASSWORD_HASH>` | `21-nodepool-config-core-password.yaml` (generate with `openssl passwd -6`, keep the single quotes) |

Apply on the hub:

```bash
oc apply -k manifests
```

Download the discovery ISO. The hub assisted-service generates it from the InfraEnv; wait for `ImageCreated`:

```bash
oc -n hcp-luks-agents wait infraenv/hcp-luks --for=condition=ImageCreated --timeout=5m
ISO_URL=$(oc -n hcp-luks-agents get infraenv hcp-luks -o jsonpath='{.status.isoDownloadURL}')
curl -kL -o discovery.iso "$ISO_URL"
```

The ISO is regenerated (new URL) whenever the InfraEnv changes, so download it again after any edit.

Boot each worker host from `discovery.iso` (BMC virtual media or USB), then approve the Agents:

```bash
oc -n hcp-luks-agents get agents
oc -n hcp-luks-agents patch agent <agent> --type merge -p '{"spec":{"approved":true}}'
```

Check the encryption on a worker:

```bash
oc debug node/<node> -- chroot /host cryptsetup status root
oc debug node/<node> -- chroot /host clevis luks list -d <root partition, e.g. /dev/sda4>
```

## Caveats

- Disk encryption happens at first boot only. Existing nodes are not encrypted in place, which is why the NodePool uses `Replace`.
- Any change to a NodePool config ConfigMap (including the `core` password) reprovisions every node in the pool.
- `tpm2: true` binds to the TPM without PCRs. To also bind to the Secure Boot state (PCR 7), replace the `clevis` block with `custom: {pin: tpm2, config: '{"pcr_bank":"sha256","pcr_ids":"7"}'}`. A firmware or Secure Boot change will then stop the disk from unlocking automatically.
- The `core` password allows console / BMC login only. SSH still requires a key. The hash is readable by anyone who can read ConfigMaps in `clusters`.
