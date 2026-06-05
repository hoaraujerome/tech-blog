+++
date = '2026-06-05T08:00:00-04:00'
title = 'How I Split AKS Node Pools and Picked VM SKUs (Part 1 of 3)'
+++

*Part 1 of a series on sizing a private AKS cluster: [Part 2 — ephemeral OS and a sizing table](/posts/aks-ephemeral-os-and-sizing-table-part-2/) · [Part 3 — Cilium overlay, IP math, and capacity](/posts/aks-cilium-overlay-ip-and-capacity-part-3/)*

I needed to size a **private AKS cluster** from scratch: not just “how many nodes,” but **which node pools**, **which Azure VM family**, and **which exact SKU** for system components, platform charts, and application workloads.

This post is what I settled on for **pool topology** and **VM family/SKU naming**. Parts 2 and 3 cover **ephemeral OS disks** and **networking/capacity**.

## The setup I was designing for

- **Private** AKS with a dedicated **system** pool and separate **user** pools
- **Platform** services (ingress, cert-manager, policy, secrets operators, etc.) — on the order of **ten Helm charts**, not ten pods
- **Application** workloads on their own pool
- **Cluster autoscaler** on each pool with different min/max per environment
- Lean **node subnet** (I only get so many VNet IPs for nodes — networking detail in Part 3)

AKS only has two pool **modes**: `System` and `User`. “Platform” is not a third mode — it is a **user** pool with labels/taints.

## Three logical tiers, two AKS modes

| Tier | AKS `mode` | What runs here | Why separate |
|------|------------|----------------|--------------|
| **System** | `System` | CoreDNS, CNI, metrics, `kube-system` | Critical path; isolate from apps |
| **Platform** | `User` + taint | Traefik, cert-manager, Gatekeeper, ESO, Reloader, … | Shared cluster services; own scale bounds |
| **Apps** | `User` | Tenant / product workloads | Blast radius and sizing independent of platform |

Microsoft recommends a [dedicated system node pool](https://learn.microsoft.com/azure/aks/best-practices-performance-scale#use-dedicated-system-node-pools) and running application work on user pools. I would not put Traefik or Gatekeeper on the system pool in production: you get **resource contention** and **shared fate** with CoreDNS and the CNI if a platform chart misbehaves.

For **sandbox**, a single pool is a tolerable shortcut. For anything that pretends to be prod, I wanted three pools.

Platform scheduling is ordinary Kubernetes:

```yaml
nodeSelector:
  agentpool: platform
tolerations:
  - key: platform
    value: "true"
    effect: NoSchedule
```

## Default VM family: general-purpose D

[Azure’s VM overview](https://learn.microsoft.com/azure/virtual-machines/sizes/overview) lists many families. For mixed Kubernetes workloads, the default is **general purpose (D-family)**:

| Family | Fit for my pools |
|--------|------------------|
| **D** | **Default** — APIs, controllers, ingress, typical microservices |
| **F** | CPU-heavy batch/encoding — only if profiling proves CPU-bound apps |
| **E** | RAM-heavy JVM/caches — second user pool later, not the first shared pool |
| **B** | Burstable — **avoid** for always-on AKS (throttling under sustained load) |
| **L** | Local-disk databases — not a generic app pool |

Microsoft’s “use **larger** node sizes to pack more pods per node” advice means **bigger D SKUs** (e.g. D8 instead of many D2s), not switching to F or E. Daemonsets and per-node agents amortize better over more schedulable pods.

My split:

- **System + platform:** `Standard_D4ds_v5` class (4 vCPU, 16 GiB)
- **Apps (prod):** `Standard_D8ds_v5` (8 vCPU, 32 GiB) for density
- **Apps (sandbox):** `D4ds_v5` is fine when cost matters

## Reading the SKU name: why `D4ds_v5` and not `D4s_v5`

Azure VM names encode features. For AKS nodes with **ephemeral OS**, the letters matter:

**`Standard_D4ds_v5`** breakdown:

| Part | Meaning |
|------|---------|
| **D** | General purpose |
| **4** | 4 vCPUs |
| **d** | **Temp / local disk** (ephemeral OS placement) |
| **s** | Premium Storage capable (remote disks) |
| **v5** | Series generation |

Common traps:

| SKU | Local temp disk | Good AKS ephemeral node? |
|-----|-----------------|--------------------------|
| `Standard_D4ds_v5` | ~150 GiB | **Yes** |
| `Standard_D4s_v5` | **None** | No — managed OS only |
| `Standard_D4as_v5` | **None** | No — AMD **diskless** ([Dasv5](https://learn.microsoft.com/azure/virtual-machines/sizes/general-purpose/dasv5-series)) |
| `Standard_D4ads_v5` | ~150 GiB | **Yes** — AMD mirror of `D4ds_v5` |

**Rule of thumb:** want ephemeral OS on local disk → look for **`d`** in the name (`dds`, `ads`).

## System pool rules I almost got wrong

I initially considered **D2** for the system pool to save money. Dedicated **system** pools have documented constraints ([manage system node pools](https://learn.microsoft.com/azure/aks/use-system-pools)):

| Requirement | Detail |
|-------------|--------|
| **VM size** | **≥ 4 vCPUs** and **≥ 4 GB** memory |
| **Min nodes** | At least **2**; **3 recommended** |
| **B-series** | **Not supported** for system pools |
| **Spot** | System pools cannot be Spot |

So **`Standard_D2ds_v5`** (2 vCPU) is out for a real system pool, even though it has enough RAM. **`Standard_D4ds_v5`** is the practical floor — not vanity sizing, **compliance with the system pool bar**.

A separate [quotas page](https://learn.microsoft.com/azure/aks/quotas-skus-regions) mentions softer limits (2 vCPU / 4 GB “might not be used”). I designed to the **stricter** system-pool document.

Prod **system min 3** pairs well with spreading across **availability zones** when the SKU and region support it.

## What I did not overthink

- **F/E families** for a first shared app pool — D is enough until metrics say otherwise
- **Ddsv6 / NVMe** for system/platform — Part 2 explains why v5 was enough for controllers
- **Putting platform on the system pool** — tempting for small clusters, wrong trade for prod

## Where this lands before Parts 2 and 3

| Pool | Mode | SKU (Intel default) | Autoscale (prod example) |
|------|------|---------------------|--------------------------|
| System | System | `Standard_D4ds_v5` | min 3 / max 5 |
| Platform | User | `Standard_D4ds_v5` | min 2 / max 5 |
| Apps | User | `Standard_D8ds_v5` | min 2 / max 5 |

Disk sizes, ephemeral OS details, and the full environment matrix are in [Part 2](/posts/aks-ephemeral-os-and-sizing-table-part-2/). [Part 3](/posts/aks-cilium-overlay-ip-and-capacity-part-3/) covers **Cilium overlay**, why a **`/26` subnet** can still work, and **quota vs capacity vs reservations**.

## References

- [Manage system node pools](https://learn.microsoft.com/azure/aks/use-system-pools)
- [VM sizes overview](https://learn.microsoft.com/azure/virtual-machines/sizes/overview)
- [AKS performance and scaling best practices](https://learn.microsoft.com/azure/aks/best-practices-performance-scale)
