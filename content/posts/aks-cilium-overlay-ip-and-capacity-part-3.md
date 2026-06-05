+++
date = '2026-06-05T08:02:00-04:00'
title = 'Cilium Overlay, Small Subnets, and Azure Capacity vs Quota (Part 3 of 3)'
+++

*Part 3 of a series on sizing a private AKS cluster: [Part 1 — node pools and VM SKUs](/posts/aks-node-pools-and-vm-sku-choice-part-1/) · [Part 2 — ephemeral OS and a sizing table](/posts/aks-ephemeral-os-and-sizing-table-part-2/)*

Parts [1](/posts/aks-node-pools-and-vm-sku-choice-part-1/) and [2](/posts/aks-ephemeral-os-and-sizing-table-part-2/) fixed **three node pools**, **Ddsv5-class SKUs**, and **ephemeral OS sizes**. I still had to answer: can a **`/26` subnet** survive **cluster autoscaler max** on all pools, and what happens when Azure runs out of **Intel VMs** in a region?

## Networking choice: Azure CNI + Cilium + overlay

I went with [**Azure CNI powered by Cilium**](https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium) in **overlay** mode:

```bash
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --pod-cidr 192.168.0.0/16 \
  ...
```

Cilium gives you the data plane (eBPF, network policies without a separate engine) and drops kube-proxy on new clusters. The important **sizing** consequence is **where pod IPs come from**.

| IPAM style | Node IPs | Pod IPs | Small node subnet (`/26`) |
|------------|----------|---------|---------------------------|
| **Overlay** + `pod-cidr` | Node subnet | **Private overlay CIDR** | **Works** for many nodes |
| Dedicated **pod subnet** | Node subnet | Separate VNet prefix | Node subnet can stay small |
| **Node subnet** (legacy) | Node subnet | **Same subnet as nodes** | **Poor** for dense clusters |

With overlay, my **`/26` is a node subnet**, not a pod subnet.

## `/26` math when all pools hit autoscaler max

Azure [reserves five addresses](https://learn.microsoft.com/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets) in every subnet.

| | Count |
|---|--------|
| Addresses in `/26` | 64 |
| Azure reserved | 5 |
| **Usable** | **59** |

From [Part 2](/posts/aks-ephemeral-os-and-sizing-table-part-2/), production worst case:

| Pool | Max nodes |
|------|-----------|
| System | 5 |
| Platform | 5 |
| Apps | 5 |
| **Total** | **15** |

**Nodes only:** `59 − 15 =` **44 IPs left** before internal load balancers, private endpoints, or other NICs in the same subnet.

Planning formula:

```text
sum(system_max, platform_max, apps_max) + other_subnet_ips ≤ 59
```

I budget **10–20 IPs** for non-node consumers → comfortable ceiling around **45 nodes total** across pools if nothing else shares the subnet. My **5+5+5** max leaves plenty of room to **raise max_count** later.

**Overlay pod CIDR** is separate — plan it for [max node count](https://learn.microsoft.com/azure/aks/concepts-network-ip-address-planning#overlay-networks) (a `/16` pod CIDR is a common starting point). Pod count does **not** consume the `/26`.

### What overlay does *not* put in the `/26`

- Pod IPs (overlay)
- Kubernetes `ClusterIP` addresses

## `max-pods` is not the same as autoscaler `max_count`

Easy to conflate when reading IP planning docs written for **flat** Azure CNI.

| Knob | Limits |
|------|--------|
| **`min_count` / `max_count`** | Number of **nodes** in the pool |
| **`max-pods`** | **Pods per node** (set at pool create) |

With **overlay**, default **max-pods** is much higher than legacy flat CNI ([maximum pods per node](https://learn.microsoft.com/azure/aks/concepts-network-ip-address-planning#maximum-pods-per-node)) — often **250** by default. Tuning `max-pods` does **not** free `/26` IPs when pods use overlay.

Dedicated **system** pools must **support at least 30 pods** per node ([use-system-pools](https://learn.microsoft.com/azure/aks/use-system-pools)) — a capability floor, not a target to run 30 system pods per node. `D4ds_v5` + overlay clears that without thinking.

You might still **lower** max-pods on system/platform for kubelet overhead (e.g. 30–50), but that is an operational choice, not a subnet lever.

## Cilium policy gotcha (Gatekeeper users)

On Azure CNI + Cilium, `NetworkPolicy` **`ipBlock` cannot match pod or node IPs** even inside a wide CIDR. Traffic to pod/node IPs may still be blocked. Prefer **`namespaceSelector`** / **`podSelector`** — relevant if you run **Gatekeeper** or hand-written policies ([Cilium FAQ](https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium#why-is-traffic-being-blocked-when-the-networkpolicy-has-an-ipblock-that-allows-the-ip-address)).

## Three gates: quota, capacity, reservations

When node creates fail, three different things get blamed — they are **not** interchangeable.

| Concept | What it is | Typical error |
|---------|------------|---------------|
| **Quota** | Subscription **permission** for vCPUs in a family/region | `QuotaExceeded` |
| **Capacity** | **Physical** slots in region/zone | `AllocationFailed`, `SKUNotAvailable` |
| **Reserved Instances** | **Prepaid billing discount** | Deploy can still fail — no capacity guarantee |

Microsoft is explicit ([Reserved VM Instances](https://learn.microsoft.com/azure/virtual-machines/prepay-reserved-vm-instances)):

> Reserved VM Instances provide a **billing discount only** and **do not reserve or guarantee compute capacity**.

[**On-demand capacity reservation**](https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview) is the mechanism that **holds hardware** for a specific VM size, region, and optional zone. You can attach it to AKS node pools at create with [`--crg-id`](https://learn.microsoft.com/azure/aks/use-capacity-reservation-groups) (user-assigned identity required; cannot retrofit an existing pool).

| Tool | Buys you |
|------|----------|
| **Quota increase** | Right to deploy up to N vCPUs |
| **Reserved Instance / Savings Plan** | Cheaper hours |
| **Capacity reservation + CRG** | Baseline nodes more likely to exist |

You can combine **RI + capacity reservation** on the same baseline slots — discount on capacity you pay to hold.

Empty capacity reservation slots **still bill**. I would only CR **prod minimums** (e.g. 3 system + 2 platform + 2 apps), not every autoscaler max.

## Regional Intel shortage: what actually helped

I hit a period of **Intel general-purpose** allocation failures in a region (Ddsv5 / Dsv5 families). Lessons:

### Mitigation order

1. **Retry availability zones** — constraints are often zonal (`--zones 1 2 3`).
2. **Stagger** pool creates/scales — do not scale system + platform + apps simultaneously during a shortage.
3. **AMD with temp disk:** `Standard_D4ads_v5` / `Standard_D8ads_v5` — not `D4as_v5` ([Part 2](/posts/aks-ephemeral-os-and-sizing-table-part-2/)).
4. **Capacity reservation** if CR creation succeeds in that region.
5. **Cobalt ARM (`*pds_v6`)** only after validating Arm64 for every platform chart.

### What did not help

- **Reserved Instances alone** — billing, not allocation.
- **Dropping system to D2** — violates **≥ 4 vCPU** system pool rule.
- **B-series** — not supported for system pools anyway.
- **`D4as_v5` as “equivalent”** — diskless; managed OS path.

Pre-flight:

```bash
az vm list-skus --location <region> --size Standard_D4ds_v5 --output table
az vm list-usage --location <region> --output table
```

## Closing the loop

| Question | Answer |
|----------|--------|
| Three pools on a `/26` with CA at max? | **Yes**, with **Cilium overlay** — 15 node IPs ≪ 59 usable |
| Room to raise `max_count`? | **~30+ nodes** headroom with conservative non-node reserve |
| Stay on Intel `D4ds_v5` during shortage? | **RI for cost**; **CR for baseline**; **AMD `ads`** for allocation; zones + retries first |
| `max-pods` vs subnet? | With overlay, tune max-pods for **density**, not `/26` |

This series is the distillation of a sizing pass I wish had been one coherent doc when I started. Your numbers will differ — but the **structure** (three pools, `d` SKUs, explicit ephemeral size, overlay for pods, three capacity gates) transferred cleanly to production planning.

## References

- [Azure CNI powered by Cilium](https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium)
- [IP address planning for AKS](https://learn.microsoft.com/azure/aks/concepts-network-ip-address-planning)
- [On-demand capacity reservation](https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview)
- [AKS capacity reservation groups](https://learn.microsoft.com/azure/aks/use-capacity-reservation-groups)
- [Upgrade capacity and cost planning](https://learn.microsoft.com/azure/aks/upgrade-capacity-cost-planning)
