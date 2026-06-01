+++
date = '2026-05-30T14:52:52-04:00'
title = 'When cert-manager Renews Your CA, Nothing Else Notices'
+++

cert-manager is excellent at keeping TLS certificates fresh—until you build a small internal PKI on top of it. Then a quiet renewal of your root CA can take down webhooks, operators, and anything that still trusts an old `ca.crt` bundled next to a perfectly valid leaf certificate.

I ran into this class of failure around **External Secrets Operator** and its validating webhook. The symptom was familiar: `CrashLoopBackOff`, logs stuck on certificate validation, cluster-wide inability to create `ExternalSecret` resources. The root cause was stranger: **the CA had been renewed, but downstream secrets had not.**

This post is what I wish I had read before debugging it—grounded in the public cert-manager discussion, not a vendor runbook.

## The setup (common and reasonable)

A typical internal-TLS stack on Kubernetes looks like this:

1. A **self-signed** `Issuer` bootstraps a **CA** `Certificate` (`isCA: true`) into a `Secret`.
2. A **CA** `Issuer` reads that secret and signs leaf certificates (webhooks, admission controllers, mTLS, etc.).
3. cert-manager writes each leaf into its own `Secret`: `tls.crt`, `tls.key`, and **`ca.crt`** (the CA that signed *that* leaf at issuance time).

External Secrets (and many other operators) follow the same pattern for their webhook TLS material. Everything works for months.

## What breaks

When the **CA certificate** is renewed, cert-manager updates the CA secret. Leaf `Certificate` objects are **not** automatically re-queued because the issuer secret changed. Worse, the leaf secret’s **`ca.crt` often stays on the old CA** even though the CA secret already contains a new one.

You can see the mismatch with fingerprints (from [cert-manager issue #5851](https://github.com/cert-manager/cert-manager/issues/5851)):

```bash
# CA secret — new fingerprint after renewal
kubectl get secret selfsigned-ca -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -fingerprint -noout

# Leaf secret — still the old CA in ca.crt
kubectl get secret selfsigned-client -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -fingerprint -noout
```

The leaf `tls.crt` may still verify cryptographically in some cases (same CA **private key**, new CA **certificate**—a subtle PKI detail), but anything that trusts **`ca.crt` from the leaf secret** can fail once that bundled CA expires. Webhooks that validate their own serving certs at startup are especially brittle: one bad bundle, infinite crash loop, cluster-wide blast radius.

**External Secrets Operator** is a sharp example: if the webhook cannot validate TLS material, you cannot reconcile `ExternalSecret` / `SecretStore` CRDs until something fixes the secrets and restarts the deployment.

## What cert-manager maintainers actually recommend

Issue [#5851](https://github.com/cert-manager/cert-manager/issues/5851) is closed as stale; there is no upstream fix that propagates a renewed CA into every dependent secret. Maintainer guidance on that thread boils down to:

**Do not treat `ca.crt` inside a leaf secret as your trust store during CA rotation.**

When a root or intermediate rotates, verifiers need a period where **both the old and new CA are trusted**. Updating only `ca.crt` in place without re-issuing leaves breaks the chain; blindly re-issuing every leaf at once risks a thundering herd and racey rollouts.

The architectural answer from the cert-manager ecosystem is **[trust-manager](https://cert-manager.io/docs/trust/trust-manager/)**: publish a **`Bundle`** (ConfigMap or Secret) that you control, and include **overlapping CA certificates** during rotation. Official docs warn against pointing a `Bundle` directly at the live cert-manager CA secret in production—rotation would instantly drop the old root from trust. The safer pattern is copying roots into a dedicated source, adding the new CA alongside the old, then removing the old when workloads have caught up.

For **leaf certificates**, the [CA issuer documentation](https://cert-manager.io/docs/configuration/ca/) is explicit: updating the CA secret **does not** trigger leaf re-issuance. You need a rotation plan and tools like **`cmctl renew`** when you intentionally rotate.

## Mitigations people use in practice (and their limits)

### Very long-lived CA

Set a long `duration` on the CA `Certificate` so renewal is rare. This is the most common operational mitigation; it is also the least “cloud-native” and it only postpones the problem.

### `renewBefore` choreography

Community workaround (also discussed on related issues): ensure the CA’s `renewBefore` is **longer than the leaf’s `duration`**, so leaves renew while the **bundled** `ca.crt` in their secret is still valid—ideally with margin (some suggest **CA `renewBefore` ≥ 2× leaf duration** so a leaf renews twice before the old bundled CA expires).

This helps **expiry alignment**; it does **not** by itself solve “CA rotated yesterday, leaf still has old `ca.crt` until next scheduled renewal.” You still need trust overlap or manual re-issuance at rotation time.

### Break-glass: delete secrets and restart

When already broken, operators often delete the CA and webhook secrets and restart the webhook deployment so cert-manager recreates a consistent chain. That is recovery, not strategy.

## A mental model that helped me

Think of three different “CA” concepts:

| Concept | Where it lives | What rotation should do |
|--------|----------------|---------------------------|
| **Signing CA** | Issuer’s secret (`tls.crt` / key) | cert-manager renews on schedule |
| **Trust anchor for verifiers** | Should be a **bundle** (trust-manager), not leaf `ca.crt` | Overlap old + new CAs |
| **Chain packaged with a leaf** | Leaf secret’s `ca.crt` | Must stay consistent with *that* leaf; won’t auto-fix when signing CA rotates |

cert-manager automates the first well. The second and third are **your** PKI design problem.

## What I would do on a new cluster today

1. **Treat CA issuers as serious PKI**—track CA expiry, document rotation, avoid “set and forget” self-signed hierarchies without trust-manager.
2. **Use trust-manager** for anything that needs to *verify* internal TLS (webhooks, clients, `--cafile` mounts)—with an explicit multi-CA bundle during rotation.
3. **Size `duration` / `renewBefore`** so leaves never outlive the CA cert stored beside them in the same secret, with slack for controller downtime.
4. **Test CA renewal in non-prod** before production learns at 3 a.m. that a webhook crash loop blocks all secret sync.

## Closing thought

cert-manager is not broken; it is honest about the limits of automating PKI rotation in a distributed system. The bug-shaped surprise is how **silent** the desync is until something validates `ca.crt` at startup and refuses to run.

If you operate internal CAs on Kubernetes—especially for admission webhooks or operators like External Secrets—plan rotation as a **trust-bundle and re-issuance** exercise, not as “cert-manager will sort it out.”

## Further reading

- [cert-manager #5851 — CA cert in Secret not updated when self-signed CA renews](https://github.com/cert-manager/cert-manager/issues/5851)
- [CA issuer docs (rotation caveats)](https://cert-manager.io/docs/configuration/ca/)
- [trust-manager documentation](https://cert-manager.io/docs/trust/trust-manager/)
