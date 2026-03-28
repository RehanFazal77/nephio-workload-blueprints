# Nephio Workload Blueprints

Upstream kpt packages for deploying 5G network functions on Kubernetes clusters managed by [Nephio](https://nephio.org). This single repository contains all blueprint packages needed for both OAI-RAN and Aether SD-Core deployments.

## What Are Blueprints?

In Nephio, a **blueprint** is a kpt package that defines a Kubernetes workload as a set of YAML manifests. Blueprints are stored in an upstream Git repository and rendered by [Porch](https://kpt.dev/guides/porch-user-guide) (the package orchestration server) into cluster-specific configurations via **PackageVariants**.

```
Blueprint (this repo)          PackageVariant (mgmt cluster)        Workload Cluster
┌──────────────────┐          ┌─────────────────────┐            ┌──────────────────┐
│ Generic YAML     │── Porch ─│ Injects cluster-    │── Porch ───│ Rendered YAML    │
│ with kpt setters │  renders │ specific values     │  publishes │ with real values │
│ (defaults)       │          │ (IPs, VLANs, names) │            │ (deployed via    │
│                  │          │                     │            │  Config Sync)    │
└──────────────────┘          └─────────────────────┘            └──────────────────┘
```

## Repository Structure

```
nephio-workload-blueprints/
│
├── cluster-baseline/                      CLUSTER FOUNDATION
│   ├── Kptfile
│   ├── namespaces.yaml                    Workload namespaces
│   ├── storage-class.yaml                 Local-path StorageClass
│   └── pod-security.yaml                  Pod security policies
│
├── platform-addons/                       PLATFORM ADDONS
│   ├── Kptfile
│   ├── monitoring/
│   │   └── metrics-server.yaml            Kubernetes Metrics Server
│   ├── storage/
│   │   └── local-path-provisioner.yaml    Rancher Local Path Provisioner
│   └── resource-management/
│       └── resource-quotas.yaml           Resource quotas
│
├── networking/                            NETWORKING
│   ├── multus-cni/                        Multus CNI + macvlan support
│   │   ├── Kptfile
│   │   ├── multus-daemonset.yaml          Multus DaemonSet (thick plugin)
│   │   ├── cni-plugins-installer.yaml     CNI plugins installer DaemonSet
│   │   ├── nad-crd.yaml                   NetworkAttachmentDefinition CRD
│   │   ├── network-crd.yaml              Network CRD
│   │   ├── nfconfig-crd.yaml             NFConfig CRD
│   │   └── nfdeployment-crd.yaml         NFDeployment CRD
│   │
│   └── vlan-init/                         VLAN Auto-Initialization
│       ├── Kptfile                        Setters: master-interface, vlan-range
│       ├── configmap.yaml                 VLAN configuration
│       ├── daemonset.yaml                 Privileged DaemonSet for VLAN creation
│       └── package-context.yaml
│
└── workloads/
    │
    ├── oai-ran/                           OAI 5G RAN
    │   ├── oai-ran-network/               Network interfaces (E1, F1, N2, N3)
    │   │   ├── Kptfile
    │   │   ├── interface-e1.yaml
    │   │   ├── interface-f1.yaml
    │   │   ├── interface-f1c.yaml
    │   │   ├── interface-f1u.yaml
    │   │   ├── interface-n2.yaml
    │   │   ├── interface-n3.yaml
    │   │   ├── interface-n4.yaml
    │   │   └── network_vpc-*.yaml
    │   │
    │   └── oai-ran/
    │       ├── oai-cp-operators/          OAI Core CP Operators (CRDs only)
    │       ├── oai-ran-operator/          OAI RAN Operator
    │       ├── pkg-example-cucp-bp/       CU Control Plane (gNB-CU-CP)
    │       ├── pkg-example-cuup-bp/       CU User Plane (gNB-CU-UP)
    │       ├── pkg-example-du-bp/         Distributed Unit (gNB-DU)
    │       ├── pkg-example-ue-bp/         UE Simulator
    │       ├── pkg-example-ue-bp-20mhz/  UE Simulator (20 MHz)
    │       └── pkg-example-ue-bp-40mhz/  UE Simulator (40 MHz)
    │
    └── sd-core/                           AETHER SD-CORE 5G
        ├── sdcore5g-cp/                   Control Plane
        │   ├── Kptfile
        │   ├── ausf/                      Authentication Server Function
        │   ├── kafka/                     Apache Kafka
        │   ├── metricfunc/                Metrics Function
        │   ├── mongodb/                   MongoDB
        │   ├── nrf/                       Network Repository Function
        │   ├── nssf/                      Network Slice Selection Function
        │   ├── pcf/                       Policy Control Function
        │   ├── udm/                       Unified Data Management
        │   ├── udr/                       Unified Data Repository
        │   └── webui/                     SD-Core WebUI
        │
        ├── sdcore5g-operator/             SD-Core Operator
        │   └── operator/
        │
        ├── pkg-example-upf-bp/            User Plane Function (UPF)
        │   ├── Kptfile
        │   ├── upfdeployment.yaml
        │   ├── interface-n3.yaml
        │   ├── interface-n4.yaml
        │   ├── interface-n6.yaml
        │   └── dnn.yaml
        │
        ├── pkg-example-amf-bp/            Access & Mobility Mgmt Function (AMF)
        │   ├── Kptfile
        │   ├── amfdeployment.yaml
        │   ├── interface-n2.yaml
        │   ├── dependency.yaml
        │   └── dnn.yaml
        │
        └── pkg-example-smf-bp/            Session Management Function (SMF)
            ├── Kptfile
            ├── smfdeployment.yaml
            ├── interface-n4.yaml
            ├── dependency.yaml
            └── dnn.yaml
```

---

## Package Details

### cluster-baseline

Foundational package applied to every workload cluster. Sets up namespaces, storage classes, and pod security policies.

**kpt Setters:**

| Setter | Default | Description |
|---|---|---|
| `cluster-name` | `ran` | Name of the target cluster |
| `cluster-type` | `workload` | Cluster type label |
| `storage-class-name` | `local-path` | StorageClass to create |

---

### platform-addons

Deploys monitoring and storage components needed by workloads.

**Components:**

- **Metrics Server** — Resource metrics (CPU/memory) for HPA and `kubectl top`
- **Local Path Provisioner** — Dynamic PV provisioning for local storage
- **Resource Quotas** — Namespace-level resource limits

---

### networking/multus-cni

Deploys [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) and related CRDs to enable multi-network pod attachments. Required for all 5G NFs that need secondary interfaces (N2, N3, E1, F1, etc.).

**Components:**

| Component | Description |
|---|---|
| Multus DaemonSet | Thick plugin mode, supports macvlan/ipvlan/bridge |
| CNI Plugins Installer | Installs macvlan, bridge, static, tuning binaries |
| CRDs | NetworkAttachmentDefinition, Network, NFDeployment, NFConfig |

---

### networking/vlan-init

**Custom package** that automates VLAN subinterface creation on workload nodes. Eliminates the need to SSH into nodes and manually create VLANs before deploying NFs.

**How it works:**

1. Deployed as a privileged DaemonSet on every node
2. Reads `master-interface` and `vlan-range` from a ConfigMap
3. Init container creates all VLAN subinterfaces and brings them UP
4. Monitor container checks every 60 seconds and recreates missing VLANs

**kpt Setters:**

| Setter | Default | Description |
|---|---|---|
| `master-interface` | `ens3` | Physical interface for VLAN subinterfaces |
| `vlan-range` | `2 3 4 5 6 7 8 9 10` | VLAN IDs to create |
| `cluster-name` | `ran` | Target cluster label |

**Example:** With `master-interface: eno49`, creates `eno49.2`, `eno49.3`, ... `eno49.10`.

---

### workloads/oai-ran

OAI 5G RAN packages from [OpenAirInterface](https://openairinterface.org/). These deploy the gNB (5G base station) split into CU-CP, CU-UP, and DU components.

| Package | Component | Interfaces |
|---|---|---|
| `oai-cp-operators` | Core CP Operators (CRDs only) | — |
| `oai-ran-operator` | RAN Operator | — |
| `oai-ran-network` | Network interfaces + NADs | E1, F1, N2, N3, N4 |
| `pkg-example-cucp-bp` | CU Control Plane | E1, F1-C, N2 |
| `pkg-example-cuup-bp` | CU User Plane | E1, F1-U, N3 |
| `pkg-example-du-bp` | Distributed Unit | F1 |
| `pkg-example-ue-bp` | UE Simulator | — |

> **Note:** `oai-cp-operators` installs only the OAI Core **operators** (for the `NFDeployment` CRD). No actual core NFs (AMF, SMF, etc.) are deployed. This enables deploying OAI-RAN independently without OAI Core.

---

### workloads/sd-core

[Aether SD-Core](https://docs.sd-core.opennetworking.org/) 5G core network packages.

| Package | Component | Interfaces |
|---|---|---|
| `sdcore5g-cp` | Control Plane (NRF, UDR, UDM, AUSF, PCF, NSSF, WebUI, MongoDB, Kafka) | — |
| `sdcore5g-operator` | SD-Core Operator | — |
| `pkg-example-upf-bp` | User Plane Function | N3, N4, N6 |
| `pkg-example-amf-bp` | Access & Mobility Mgmt Function | N2, SBA |
| `pkg-example-smf-bp` | Session Management Function | N4, SBA |

> **Important:** AMF and SMF have `Dependency` CRs referencing UPF. UPF must be Published first, or AMF/SMF will remain in Draft.

---

## kpt Function Images

All packages use publicly accessible kpt function images from GHCR:

| Function | Image |
|---|---|
| `apply-setters` | `ghcr.io/kptdev/krm-functions-catalog/apply-setters:v0.2` |
| `apply-replacements` | `ghcr.io/kptdev/krm-functions-catalog/apply-replacements:v0.1.1` |
| `set-namespace` | `ghcr.io/kptdev/krm-functions-catalog/set-namespace:v0.4.1` |

> **Why not GCR?** The original `gcr.io/kpt-fn/*` images were deprecated and now require authentication. The GHCR mirrors at `ghcr.io/kptdev/krm-functions-catalog/*` are publicly accessible and functionally identical.

---

## How Packages Are Consumed

This repository is registered as an upstream Porch repository on the Nephio management cluster:

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: nephio-workload-blueprints
spec:
  type: git
  content: Package
  git:
    repo: https://github.com/YOUR-ORG/nephio-workload-blueprints.git
    branch: main
    directory: /
```

PackageVariants then reference packages by their directory path:

```yaml
apiVersion: config.porch.kpt.dev/v1alpha1
kind: PackageVariant
metadata:
  name: vlan-init-ran
spec:
  upstream:
    repo: nephio-workload-blueprints
    package: networking/vlan-init          # ← directory path relative to repo root
  downstream:
    repo: ran
    package: vlan-init
  pipeline:
    mutators:
      - image: ghcr.io/kptdev/krm-functions-catalog/apply-setters:v0.2
        configMap:
          master-interface: ens3           # ← overrides Kptfile default
```

For the full deployment configuration, see the companion repository: [nephio-deployment-mgmt](../nephio-deployment-mgmt/).

---

## Adding a New Package

To add a new blueprint package:

1. Create a directory under the appropriate category (`networking/`, `workloads/`, etc.)
2. Add a `Kptfile` with package metadata and pipeline mutators
3. Add your Kubernetes YAML manifests
4. Use `# kpt-set: ${setter-name}` comments for values that should be customizable per cluster
5. Push to this repository
6. Create a PackageVariant in `nephio-deployment-mgmt` referencing your new package

---

## License

Refer to the individual upstream projects for their respective licenses:

- [Nephio](https://github.com/nephio-project) — Apache 2.0
- [OpenAirInterface](https://gitlab.eurecom.fr/oai/openairinterface5g) — OAI Public License
- [Aether SD-Core](https://github.com/omec-project) — Apache 2.0
- [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) — Apache 2.0
