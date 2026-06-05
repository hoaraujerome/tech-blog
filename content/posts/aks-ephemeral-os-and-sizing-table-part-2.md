+++
date = '2026-06-05T08:01:00-04:00'
title = 'Ephemeral OS and a Concrete AKS Sizing Table (Part 2 of 3)'
+++

*Part 2 of a series on sizing a private AKS cluster: [Part 1 — node pools and VM SKUs](/posts/aks-node-pools-and-vm-sku-choice-part-1/) · [Part 3 — Cilium overlay, IP math, and capacity](/posts/aks-cilium-overlay-ip-and-capacity-part-3/)*

In [Part 1](/posts/aks-node-pools-and-vm-sku-choice-part-1/) I split the cluster into **system**, **platform**, and **apps** pools and picked **D-family** SKUs with a **`d`** for local temp disk. This post is the **disk strategy**, the **full sizing table**, and the **Intel vs AMD vs v6** decisions that fed it.

## Why ephemeral OS on AKS

AKS prefers [**ephemeral OS disks**](https://learn.microsoft.com/azure/aks/concepts-storage) when the VM SKU allows: the OS volume lives on **local** temp storage instead of a remote managed disk. You get faster reimage/scale and lower latency for kubelet and image layers.

Relevant knobs:

| Setting | What I use |
|---------|------------|
| `osDiskType` | `Ephemeral` |
| `kubeletDiskType` | `OS` — images and kubelet data share the OS disk (simple) |
| `osDiskSize` / `node-osdisk-size` | **Set explicitly** per pool |

### Set `osDiskSize` on purpose

On SKUs with large temp disks (150–300 GiB), leaving size unset can lead Azure/AKS to allocate a **much larger** ephemeral OS than you need — especially as defaults evolve on big temp SKUs.

I sized intentionally:

| Pool | `osDiskSize` | Why |
|------|--------------|-----|
| **System** | **64 GiB** | Few images; CoreDNS/CNI only |
| **Platform** | **128 GiB** | ~10 Helm charts, more image layers |
| **Apps** | **128 GiB** | Pull-heavy apps; bump toward 150 only if image churn proves it |

Each cap must stay **≤ the SKU max temp** (150 GiB on D4ds_v5 / D4ads_v5; 300 GiB on D8ds_v5 / D8ads_v5).

Ephemeral is for **stateless nodes**. Anything that must survive node loss belongs on **PersistentVolumes**, not the OS disk.

## Ddsv5 vs Ddsv6: I stayed on v5

I compared **`Standard_D4ds_v5`** and **`Standard_D4ds_v6`**. On paper, v6 wins local IOPS (NVMe temp vs SCSI temp on v5). For **system and platform** pools, I did not pay the v6 premium.

Two reasons:

1. **Cost** — roughly tens of dollars per node per month adds up across three pools.
2. **Ephemeral OS on NVMe** — Microsoft documents that with ephemeral OS on NVMe VMs, the OS path may not expose “full NVMe” performance the way raw local disk benchmarks suggest ([NVMe temp FAQs](https://learn.microsoft.com/azure/virtual-machines/enable-nvme-temp-faqs)). On a single-disk D4 class VM using the whole disk for OS + `kubeletDiskType: OS`, the headline 75k IOPS are often **misleading** for day-to-day node I/O.

**v6** is still reasonable if you **standardize the whole fleet** on v6 or v5 is unavailable in your region. It was not worth it **only** for Traefik and cert-manager.

## AMD fallback: `ads`, not `as`

When Intel **Ddsv5** hits regional capacity limits, Microsoft often points to **AMD v5**. The correct mirror SKU includes **`d`**:

| Intel | AMD fallback | Local temp |
|-------|--------------|------------|
| `Standard_D4ds_v5` | **`Standard_D4ads_v5`** | 150 GiB |
| `Standard_D8ds_v5` | **`Standard_D8ads_v5`** | 300 GiB |

**`Standard_D4as_v5`** is **not** equivalent. [Dasv5](https://learn.microsoft.com/azure/virtual-machines/sizes/general-purpose/dasv5-series) is **diskless by design** — your pricing calculator row showing **N/A** for local storage is the tell. AKS falls back to **managed** Premium OS disks. Fine as a **short-term** capacity hack; it breaks an ephemeral-first design.

I learned this the hard way after misreading “AMD alternative” as `D4as_v5` instead of `D4ads_v5`.

Pre-flight:

```bash
az vm list-skus --location <region> --size Standard_D4ads_v5 --output table
```

## Reference sizing table

Template from my design exercise — adjust for your workloads and region.

### SKUs and disks

| Pool | AKS mode | SKU | OS disk (ephemeral) |
|------|----------|-----|---------------------|
| System | System | `Standard_D4ds_v5` | 64 GiB |
| Platform | User + taint | `Standard_D4ds_v5` | 128 GiB |
| Apps (prod) | User | `Standard_D8ds_v5` | 128 GiB |
| Apps (sandbox) | User | `Standard_D4ds_v5` | 128 GiB |

AMD: swap `ds` → `ads` at the same size tier.

### Autoscale bounds

| Pool | Production | Non-prod | Sandbox |
|------|------------|----------|---------|
| System | min **3** / max **5** | min **2** / max **5** | min **2** / max **5** |
| Platform | min **2** / max **5** | min **2** / max **5** | min **1** / max **5** |
| Apps | min **2** / max **5** | min **2** / max **5** | min **1** / max **5** |

Operational habits:

- Set **`node_count` = `min_count`** when turning on the cluster autoscaler.
- Sandbox **min 1** on platform/apps saves money; you accept no HA during node drains.

### Worst-case node count (prod, all pools at max)

```text
5 + 5 + 5 = 15 nodes
```

With **Cilium overlay** (Part 3), that number drives **node subnet** IPs — not pod IPs.

### Rough vCPU quota ask

| Pool | Max nodes | vCPU each | Subtotal |
|------|-----------|-----------|----------|
| System | 5 | 4 | 20 |
| Platform | 5 | 4 | 20 |
| Apps | 5 | 8 | 40 |
| **Total** | 15 | | **80** |

Add **upgrade surge** headroom (~one extra node per pool during rolling upgrade) → I asked for **~100 vCPUs** in the Ddsv5 family when requesting quota.

## Decisions I would make again

| Choice | Rationale |
|--------|-----------|
| D4 system (not D2) | [System pool ≥ 4 vCPU](https://learn.microsoft.com/azure/aks/use-system-pools) |
| D4 platform (not D8) | Controllers + ingress, not app-scale density |
| D8 apps in prod | Fewer, larger nodes for daemonset overhead |
| v5 over v6 | Cost; NVMe upside muted for ephemeral OS on controllers |
| 64 vs 128 GiB OS | Explicit caps; avoid “use all temp” surprises |

## What’s next

[Part 3](/posts/aks-cilium-overlay-ip-and-capacity-part-3/) answers the question that kept me up at night with a **`/26` node subnet**: how do fifteen autoscaled nodes and hundreds of pods fit? Short answer: **overlay + Cilium** — pods do not eat your `/26`. I also unpack **`max-pods` vs autoscaler `max_count`**, and **quota vs Reserved Instances vs on-demand capacity reservation** when a region runs out of Intel SKUs.

## References

- [Storage options in AKS](https://learn.microsoft.com/azure/aks/concepts-storage)
- [Ephemeral OS disks for Azure VMs](https://learn.microsoft.com/azure/virtual-machines/ephemeral-os-disks)
- [Dadsv5 series](https://learn.microsoft.com/azure/virtual-machines/sizes/general-purpose/dadsv5-series)
