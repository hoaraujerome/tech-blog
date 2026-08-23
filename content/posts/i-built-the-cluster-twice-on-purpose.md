+++
date = '2026-08-23T13:10:00-04:00'
draft = true
title = 'I Built the Kubernetes Cluster Twice on Purpose'
+++

Last year I stood up a Kubernetes cluster on AWS from scratch — kubeadm, Terraform,
Packer, Ansible, Cilium, private nodes, EC2 Instance Connect instead of a bastion. I
wrote about the [design constraints](https://blog.hoaraujerome.com/posts/aws-kubernetes-homelab-design-decisions/)
and the [first-boot automation](https://blog.hoaraujerome.com/posts/automating-kubeadm-init-and-join-on-aws-my-cloud-homelab-approach/).
That work lives in [k8s-homelab](https://github.com/hoaraujerome/k8s-homelab). It did what
I needed: CKA prep, multi-cloud fluency, a real two-node topology with control plane and
worker joined over SSM.

This year I did it again — and I did **not** fork the homelab repo.

[Phase 2](https://github.com/hoaraujerome/vllm-deployment/blob/main/phase2/README.md) of
[vllm-deployment](https://github.com/hoaraujerome/vllm-deployment) is greenfield code
under a new tree, with a different destination and a different definition of "minimal."
Same family of tools. Same AWS region and instance size. Same philosophical allergy to
managed Kubernetes. Different product.

That choice was deliberate. This post is why.

## The first build was for learning breadth

The homelab was optimized for **depth on the cluster itself**. Two nodes so join logic
mattered. SSM so the control plane could hand a token to a worker without me in the loop.
One shared AMI with role branching at boot — more moving parts, more representative of how
a small real cluster behaves.

I was not trying to get somewhere quickly. I was trying to **see the machinery**.

That succeeded. I trust kubeadm-first-boot now. I trust EICE for private nodes. I have
opinions about Cilium that come from having installed it the hard way.

But homelab carries homelab assumptions: Ubuntu 24.04 because that is what I pinned then,
provider versions from that era, layout choices that made sense for certification study
and cost posts — not for a phased path toward running vLLM on the same footprint.

When I started vllm-deployment, the tempting move was obvious: clone what works, change
the model name, call it Phase 2.

I rejected that.

## The second build was for a destination

vllm-deployment is structured in phases. Phase 1 was local inference on Apple Silicon.
Phase 2 is the cluster factory. Phase 4 is vLLM on the cluster. Phase 3 is WireGuard so
my laptop can run Helm without SSH acrobatics.

That arc changes what "minimal" means.

Phase 2 does not need two nodes. It needs **one schedulable node** that survives reboot
and passes objective checks. It does not need SSM choreography yet — there is no worker
to join. It does not need vLLM, GPU, or ingress baked in prematurely. It needs a
**contract**: infrastructure, bootstrap, node Ready, smoke workload — provable before the
LLM story starts.

Homelab taught me patterns. vllm-deployment needed **constraints written for a different
finish line**.

## Inspiration is not a fork

I told every agent session the same thing: k8s-homelab is **reference architecture**, not
source to import. Greenfield paths. Frozen reject lists. Do not copy the Ubuntu pin. Do
not copy the AWS provider pin. Do not merge Terraform state between the Packer builder
network and the cluster network. Do not paste the multi-node bootstrap script when the
spec says single-node.

Agents love continuity. Mention a repo once and the next diff imports half of it. That is
efficient if your goal is speed. It is hazardous if your goal is **clarity about what this
phase owns**.

Rebuilding let me keep what I trust — AMI first-boot, EICE, Cilium, kubeadm built-in PKI
— and drop what Phase 2 does not owe yet: second node, SSM, homelab's directory history,
certification-shaped complexity.

The cost is real: I typed Terraform twice. I maintained two repos. I resisted the voice
that says duplication is always waste.

The benefit is also real: Phase 2's README is the contract for Phase 2. Not a homelab
README with strikethroughs. Not a fork drifting out of sync while I pretend it is a new
project.

## What I changed on purpose

Some differences are cosmetic. Some are judgment calls I expect to matter later.

**Topology.** One combined control-plane-and-worker node for the MVP. Homelab's two-node
split was the right teacher; single-node is the right factory for a solo builder heading
toward vLLM on one box first.

**Bootstrap coordination.** Homelab uses SSM between nodes. Phase 2 runs kubeadm init and
Cilium install on first boot with no join path — simpler script, fewer failure modes while
the validation ladder hardens.

**Versions and pins.** Newer Kubernetes, newer Cilium line, Ubuntu 26.04 LTS arm64, AWS
provider 6.x — chosen for the greenfield repo, not inherited because the old pin was
"good enough" last summer.

**Orchestration.** A Makefile and a gate script define how work runs and when Phase 2 is
done. Homelab grew more organically; vllm-deployment was built with
[loop engineering](/posts/done-when-the-script-says-so-not-the-agent/)
in mind from the start.

**Scope discipline.** No vLLM in the AMI. No GPU. No ingress in Phase 2. The cluster had
to become boring infrastructure before it became an inference platform.

None of that required throwing homelab away. It required **not pretending homelab was
already the answer**.

## When copying would have been the wrong kind of fast

Forking would have saved days. I am not pretending otherwise.

It would also have imported decisions I would have had to un-make in prose — "we don't
need the worker yet but the module still provisions it," "ignore the SSM path," "this
state file layout is historical." Technical debt disguised as momentum.

Greenfield was slower upfront and cleaner at the Phase 2 finish line: full ladder green,
reboot survived, kubelet enabled, smoke pod running — on code that only claims to do that
much.

I have built one Kubernetes cluster twice. The second time was not because I forgot how.
It was because the **first** answered "how does this work?" and the **second** had to
answer "does this path get me to vLLM without lying?"

Homelab remains my learning lab. vllm-deployment is the productized path. Both can coexist
in my GitHub without one pretending to be the other.

Phase 3 is next. I already know which repo holds the constraints — and which one I will
not fork to get there.
