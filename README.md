# HCP cluster with TPM2 disk encryption

Manifests for a Hosted Control Planes (HyperShift) cluster on the Agent platform (bare metal).
The worker NodePool uses MachineConfigs to:

- encrypt the RHCOS root disk with LUKS, unlocked automatically by the host TPM 2.0 (Clevis)
- set a password for the `core` user (console login)

Target: OpenShift 4.22 (release image `4.22.15` in `10-hostedcluster.yaml` and `22-nodepool.yaml`, keep both identical), applied on the hub (MCE / ACM) cluster.

## Layout

| File | What it does |
|---|---|
| `manifests/00-namespaces.yaml` | `clusters` (HostedCluster, NodePool, config ConfigMaps). InfraEnv, NMStateConfig, Agents and the worker VM go in the agent namespace `<AGENT_NAMESPACE>`, which must already exist. The control plane namespace `clusters-hcp-luks` is created by HyperShift. |
| `manifests/01-secrets.yaml` | Pull secret (in both namespaces) and SSH public key. |
| `manifests/02-infraenv.yaml` | Discovery ISO for the workers. Labels every Agent `hcp-luks.example.com/tpm2: "true"`, and sets a `core` console password on the discovery image (`ignitionConfigOverride`). |
| `manifests/03-nmstateconfig.yaml` | NMStateConfig for the single worker: static IPv4, gateway, DNS, matched to the host by NIC MAC. Embedded in the ISO via the InfraEnv `nmStateConfigLabelSelector` and kept on the installed node. |
| `manifests/04-agent-rbac.yaml` | Role `capi-provider-role` in the agent namespace. HyperShift binds the Agent CAPI provider to it; without it the HostedCluster never creates its control plane. |
| `manifests/10-hostedcluster.yaml` | Hosted control plane: release, networking, managed etcd, service publishing (API via LoadBalancer, others via Route). |
| `manifests/20-nodepool-config-luks-tpm2.yaml` | ConfigMap with MachineConfig `99-worker-luks-tpm2`: LUKS on `/dev/disk/by-partlabel/root`, Clevis `tpm2: true`, root re-created as XFS on `/dev/mapper/root`. |
| `manifests/21-nodepool-config-core-password.yaml` | ConfigMap with MachineConfig `99-worker-core-password`: sets `passwordHash` for user `core`. |
| `manifests/22-nodepool.yaml` | Worker NodePool: selects TPM-labelled Agents, references both ConfigMaps in `spec.config`, upgrade type `Replace`. |
| `manifests/kustomization.yaml` | Applies everything in order. |
| `vm/01-iso-datavolume.yaml` | DataVolume importing the discovery ISO from the InfraEnv download URL. |
| `vm/02-worker-vm.yaml` | Worker VM (OpenShift Virtualization on the hub), created stopped: persistent vTPM, persistent UEFI with Secure Boot, bridged NAD (MAC assigned by kubemacpool), root disk + discovery ISO. |
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
| `<AGENT_NAMESPACE>` | `01-secrets.yaml`, `02-infraenv.yaml`, `03-nmstateconfig.yaml`, `04-agent-rbac.yaml`, `10-hostedcluster.yaml`, `vm/*.yaml` |
| `<PULL_SECRET_JSON>` | `01-secrets.yaml` |
| `<SSH_PUBLIC_KEY>` | `01-secrets.yaml`, `02-infraenv.yaml` |
| `<WORKER0_MAC>`, `<WORKER0_IP>`, `<PREFIX_LENGTH>`, `<GATEWAY>`, `<DNS_SERVER>` | `03-nmstateconfig.yaml` (copy the document to add hosts) |
| `<BASE_DOMAIN>`, `<WORKER_MACHINE_CIDR>` | `10-hostedcluster.yaml` |
| `<CORE_PASSWORD_HASH>` | `21-nodepool-config-core-password.yaml` (installed node) and `02-infraenv.yaml` (discovery image, inside the `ignitionConfigOverride` JSON) (generate with `openssl passwd -6`, keep the single quotes) |
| `<NAD_NAMESPACE>/<NAD_NAME>` | `vm/02-worker-vm.yaml` (VM flow only) |

Storage classes default to the ODF ones (`ocs-storagecluster-ceph-rbd*`); change them if your hub uses others.

The commands below use the agent namespace from a shell variable:

```bash
AGENT_NS=<AGENT_NAMESPACE>
```

Apply on the hub:

```bash
oc apply -k manifests
```

Download the discovery ISO. The hub assisted-service generates it from the InfraEnv; wait for `ImageCreated`:

```bash
oc -n "$AGENT_NS" wait infraenv/hcp-luks --for=condition=ImageCreated --timeout=5m
ISO_URL=$(oc -n "$AGENT_NS" get infraenv hcp-luks -o jsonpath='{.status.isoDownloadURL}')
curl -kL -o discovery.iso "$ISO_URL"
```

The ISO is regenerated (new URL) whenever the InfraEnv or a matching NMStateConfig changes, so download it again after any edit. The host applies the NMStateConfig whose MAC matches its NIC.

Boot the worker host from `discovery.iso` (BMC virtual media or USB).

### Worker as a VM on the hub (OpenShift Virtualization)

The VM MAC is not known in advance, so the order differs from the bare metal flow: create the VM first (stopped), read its MAC, then build the ISO.

Prerequisites: fill `<NAD_NAMESPACE>/<NAD_NAME>` (bridge or localnet NAD on the machine network) in `vm/02-worker-vm.yaml`, and set `spec.vmStateStorageClass` in the HyperConverged CR (needed by the persistent vTPM).

1. Create the namespaces and the VM (`runStrategy: Halted`). kubemacpool assigns the MAC and writes it into the VM spec:

   ```bash
   oc apply -f manifests/00-namespaces.yaml
   oc apply -f vm/02-worker-vm.yaml
   oc -n "$AGENT_NS" get vm hcp-luks-worker-0 \
     -o jsonpath='{.spec.template.spec.domain.devices.interfaces[0].macAddress}{"\n"}'
   ```

   An empty result means kubemacpool is disabled for the namespace: set `macAddress` in the VM by hand instead.

2. Put that MAC in both `<WORKER0_MAC>` fields of `manifests/03-nmstateconfig.yaml`, then apply everything and wait for the ISO:

   ```bash
   oc apply -k manifests
   oc -n "$AGENT_NS" wait infraenv/hcp-luks --for=condition=ImageCreated --timeout=5m
   ```

3. Import the ISO (the URL carries a token: it is injected at apply time and never written to the repo), then start the VM:

   ```bash
   ISO_URL=$(oc -n "$AGENT_NS" get infraenv hcp-luks -o jsonpath='{.status.isoDownloadURL}')
   sed "s|<ISO_URL>|${ISO_URL}|" vm/01-iso-datavolume.yaml | oc apply -f -
   oc -n "$AGENT_NS" wait dv/hcp-luks-discovery-iso --for=condition=Ready --timeout=15m
   virtctl -n "$AGENT_NS" start hcp-luks-worker-0
   ```

Do not re-apply `vm/02-worker-vm.yaml` afterwards: it resets `runStrategy` to `Halted` and stops the VM.

Then approve the Agent:

```bash
oc -n "$AGENT_NS" get agents
oc -n "$AGENT_NS" patch agent <agent> --type merge -p '{"spec":{"approved":true}}'
```

Check the encryption on a worker:

```bash
oc debug node/<node> -- chroot /host cryptsetup status root
oc debug node/<node> -- chroot /host clevis luks list -d <root partition, e.g. /dev/sda4>
```

## Caveats

- Disk encryption happens at first boot only. Existing nodes are not encrypted in place, which is why the NodePool uses `Replace`.
- Any change to a NodePool config ConfigMap (including the `core` password) reprovisions every node in the pool. With a single worker and no spare Agent, the node is removed first: workloads are down until the host is reinstalled.
- Single worker: the HostedCluster uses `infrastructureAvailabilityPolicy: SingleReplica` so guest infra components (ingress router, etc.) run with one replica.
- `tpm2: true` binds to the TPM without PCRs. To also bind to the Secure Boot state (PCR 7), replace the `clevis` block with `custom: {pin: tpm2, config: '{"pcr_bank":"sha256","pcr_ids":"7"}'}`. A firmware or Secure Boot change will then stop the disk from unlocking automatically.
- The `core` password allows console / BMC login only. SSH still requires a key. The hash is readable by anyone who can read ConfigMaps in `clusters`.
