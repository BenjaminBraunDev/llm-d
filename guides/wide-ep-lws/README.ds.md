# Wide Expert Parallelism with DisaggregatedSet

This variant deploys the same DeepSeek-R1-0528 wide-EP prefill/decode stack as the [main guide](./README.md), but manages both multi-host roles with a single [DisaggregatedSet](https://lws.sigs.k8s.io/docs/concepts/disaggregatedset/) resource from [LeaderWorkerSet (LWS)](https://lws.sigs.k8s.io/) instead of two independently managed LeaderWorkerSets.

For the wide-EP architecture itself (DP/EP layout, DeepEP, RDMA requirements, hardware backends), refer to the **[main guide](./README.md)** — nothing about the serving stack changes here. For DisaggregatedSet concepts and operations, the **[P/D Disaggregation DS variant](../pd-disaggregation/README.ds.md)** covers the shared material in more depth.

## Why DisaggregatedSet here?

The main guide already uses LeaderWorkerSet for each role, so the multi-host mechanics (leader election, `LWS_LEADER_ADDRESS`, rank env vars, group-atomic restarts) are identical. What changes is the layer above:

* **One declarative unit** — prefill and decode live in one custom resource with a single revision spanning both roles, instead of two LWS objects whose versions and rollouts you coordinate by hand.
* **Coordinated rollouts** — a template change (image bump, flag change) rolls prefill and decode as one version-synchronized unit per slice, never leaving a new-image prefill talking to an old-image decode.
* **Slices** — `spec.slices` stamps out N independent copies of the *entire* 32-GPU topology. With this guide's default shape, each slice is 2 multi-host groups (16-GPU prefill + 16-GPU decode). Changing `slices` is a pure scale operation: existing slices are never re-rolled.

> [!NOTE]
> The llm-d Router (EPP) discovers all DP ranks by pod label as flat pools and is not slice-aware: it may pair a prefill in one slice with a decode in another. KV transfer between slices must therefore remain possible on your fabric. This is the same caveat as the P/D DS variant, and it matters more here if you use placement to confine slices to separate RDMA domains.

## Prerequisites

1. Complete the **[Prerequisites](./README.md#prerequisites)** of the main guide (client tools, repo, env vars, HF token secret, and — for the DeepEP path — a cluster with full-mesh RDMA connectivity).
2. Install the LWS controller with the DisaggregatedSet API enabled. The `slices` field used here was added after LWS `v0.9.0` — install `v0.10.0` or newer:

   ```bash
   export LWS_CHART_VERSION=0.10.0
   helm install lws oci://registry.k8s.io/lws/charts/lws \
     --version=${LWS_CHART_VERSION} \
     --namespace lws-system \
     --create-namespace \
     --set enableDisaggregatedSet=true \
     --wait --timeout 300s
   kubectl get crd disaggregatedsets.disaggregatedset.x-k8s.io
   ```

Use the same environment variables as the main guide (`GUIDE_NAME="wide-ep-lws"`, `NAMESPACE`, etc.).

## Installation

### 1. Deploy the llm-d Router

Follow the **[gateway/router steps of the main guide](./README.md#installation)** unchanged. The EPP discovers model server pods (and their per-DP-rank ports) by label, which the DisaggregatedSet's pod templates carry verbatim, so routing — including DP-aware per-rank scoring — is identical.

### 2. Deploy the Model Server

On GKE (A3 Ultra / H200-class nodes with RoCE):

```bash
kubectl apply -n ${NAMESPACE} -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/gpu/vllm-ds/gke
```

On providers where you manage RDMA wiring yourself, start from `modelserver/gpu/vllm-ds/base` and add a provider overlay, mirroring how the main guide's provider overlays patch `modelserver/gpu/vllm/base`.

This creates one `DisaggregatedSet` named `wide-ep-llm-d`. The controller fans it out into one LeaderWorkerSet per `(slice, role)`, named `<ds>-<slice>-<revision>-<role>`, each an exact equivalent of the main guide's hand-managed LWS:

```bash
kubectl get leaderworkerset -n ${NAMESPACE} -l disaggregatedset.x-k8s.io/name=wide-ep-llm-d

# NAME                              READY   DESIRED   UP-TO-DATE   AGE
# wide-ep-llm-d-0-6c8d4f9b-decode    1       1         1           15m
# wide-ep-llm-d-0-6c8d4f9b-prefill   1       1         1           15m
```

The revision hash in the name changes on each rollout — always query children through the `disaggregatedset.x-k8s.io/{name,slice,role}` labels, never hardcoded names.

## Operating

### Scale slices (whole 32-GPU copies)

```bash
kubectl patch disaggregatedset wide-ep-llm-d -n ${NAMESPACE} --type merge -p '{"spec":{"slices":2}}'
```

Each additional slice is a complete prefill+decode copy — with the default shape, 32 more GPUs across 4 more multi-host groups. The new slice comes up at the current revision without touching existing slices. Scaling down removes the highest-indexed slices.

### Scale replicas within a role

Per-role `replicas` applies **per slice** and adds whole multi-host groups (16 GPUs each with the default shape):

```bash
kubectl patch disaggregatedset wide-ep-llm-d -n ${NAMESPACE} --type json \
  -p '[{"op": "replace", "path": "/spec/roles/0/spec/replicas", "value": 2}]'
```

### Rolling updates

Any pod-template change creates a new revision; each slice rolls to it independently while keeping a complete same-version P/D set serving. With wide-EP group sizes, remember that a surge requires a full spare group's worth of accelerators (16 GPUs by default) — see Known Issues in the [P/D DS variant](../pd-disaggregation/README.ds.md#known-issues), which apply here with larger units.

### Placement

> [!WARNING]
> `spec.placementPolicy` is **not yet merged or released** in LWS (tracked in [lws#848](https://github.com/kubernetes-sigs/lws/issues/848)); released controllers reject the field. A commented example ships in `vllm-ds/base/disaggregatedset.yaml`.

Once available, placement policy can confine each slice's all-to-all and KV-transfer traffic to a single RDMA domain (for example a GKE topology block) and spread slices across domains.

## Verification and Benchmarking

Identical to the **[main guide](./README.md)** — same endpoint, model name, and inference-perf profile. Confirm both roles serve by checking logs per role label:

```bash
kubectl logs -n ${NAMESPACE} -l llm-d.ai/role=prefill --tail=10
kubectl logs -n ${NAMESPACE} -l llm-d.ai/role=decode --tail=10
```

## Cleanup

Deleting the DisaggregatedSet cascades to all child LeaderWorkerSets, pods, and per-slice Services:

```bash
kubectl delete -n ${NAMESPACE} -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/gpu/vllm-ds/gke
```

## Validation status

The DisaggregatedSet mechanics of this variant (multi-host role fan-out, DP-rank port routing through the DS-generated names, slice scaling, coordinated rollouts) were validated on GKE H200 (A3 Ultra, RoCE) at reduced scale. Full-scale 32-GPU-per-slice validation runs on the maintainers' E2E clusters alongside the main guide's pipelines.

## Further Reading

* [P/D Disaggregation with DisaggregatedSet](../pd-disaggregation/README.ds.md) — shared DS concepts, operations, and known issues
* [DisaggregatedSet API reference](https://lws.sigs.k8s.io/docs/reference/disaggregatedset.v1/)
* [KEP-846: DisaggregatedSet Slices](https://github.com/kubernetes-sigs/lws/tree/main/keps/846-disaggregatedset-slices)
