+++
date = '2026-09-13T13:00:00-04:00'
title = 'The Cluster Lived on the Node. I Needed It on My Laptop.'
+++

I finished [Phase 3](https://github.com/hoaraujerome/vllm-deployment/blob/main/phase3/README.md) of the same
project that gave me [Phase 2](https://github.com/hoaraujerome/vllm-deployment/blob/main/phase2/README.md) —
and the post about [when the script says done, not the agent](https://blog.hoaraujerome.com/posts/done-when-the-script-says-so-not-the-agent/).
Phase 2 proved the cluster exists: private node, kubeadm, Cilium, smoke workload, survives
reboot. Phase 3 proved something narrower and, for what comes next, more important: I can
admin that cluster from my laptop over WireGuard, without opening an EC2 Instance Connect
tunnel every time I want to run kubectl.

That sounds like a small upgrade. It did not feel small while building it.

## Two contracts, not one big phase

The mistake I wanted to avoid was treating "make Kubernetes work" and "make Kubernetes
usable from where I actually sit" as the same job.

Phase 2 is the factory. VPC, NAT, a single node, break-glass SSH through Instance Connect,
bootstrap on the instance, validation on the node. That contract is precious. I spent
weeks freezing it so an agent would not quietly reopen AMI pins, Cilium versions, or
bootstrap paths every time I asked for something adjacent.

Phase 3 is the access layer. A dedicated WireGuard box in an existing public subnet, split
tunnel so only the VPC crosses the VPN, security group rules so the API server accepts
traffic from the VPN, kubeconfig on the laptop, a check script that proves the tunnel is
up and kubectl works from home. No vLLM. No ingress. No reason to touch the cluster
factory.

Keeping that boundary written down mattered more than any module layout. The agent's
default move is consolidation: put WireGuard on the node, bake VPN into the AMI, fold
laptop kubectl gates into the Phase 2 script because it is "just one more check." Each of
those shortcuts would have made the demo shorter and the project harder to reason about
later. Phase 3's README is mostly a reject list with a WireGuard EC2 at the end.

## What "done" meant this time

I reused the discipline from Phase 2, but the ladder changed shape.

Provisioning gates still exist: plan, apply, Trivy, the usual Terraform hygiene. Runtime
gates are what Phase 3 is really about: WireGuard interface up, kubectl get nodes from the
laptop, and a deliberate rule that Phase 3 scripts must not call Instance Connect for
routine work. Break-glass stays in Phase 2. Daily ops move to the VPN.

The agent declared victory several times on partial evidence. The tunnel pinged. curl to
the API port returned JSON from inside the VPC. The Terraform apply succeeded. None of
that was the contract. The contract was make check-full exiting zero with WireGuard
connected and kubectl answering from a kubeconfig that lived on my machine, not on the
node.

Same principle as Phase 2. Different proof.

## Surprises that were not in the README

The expensive line item was not the VPN. A t4g.nano for WireGuard is noise next to a NAT
Gateway that exists so a private node can pull images. I knew NAT was there; I had not
internalized that it dominates the monthly bill while the access layer I was celebrating
costs less than a coffee. That is a homelab lesson I will carry into Phase 4 planning,
not a pricing guide — just the kind of external-feedback preview Andrew Ng's loops keep
pointing at: reality that does not care how clever the VPN setup was.

Terraform wanted to replace the WireGuard instance on every plan for a while, because a
provider quirk around elastic IPs and a launch-time flag turns into perpetual destroy-and-
recreate unless you know to stop fighting the state machine. The agent proposed apply
again. The script said the plan should be empty. I believed the script.

Fetching kubeconfig sounded boring until it wasn't. The break-glass path through Instance
Connect was fragile on my laptop; the WireGuard hop needed TCP forwarding, not the SSH
pattern that runs the second login on the jump host. A log line meant for humans landed
inside the kubeconfig file once because stdout was not stderr. Each issue is a paragraph
in a troubleshooting doc somewhere. Collectively they are the texture of trusting a new
access path: the cluster was fine; my path to it was not yet believable.

## What I trust now

I trust that Phase 2 and Phase 3 can evolve on different clocks. Cluster health still
means make check-cluster from the Phase 2 tree. Laptop admin means WireGuard on, then
kubectl from home. If the VPN is broken, I have not lost the cluster — I have lost
convenience, and Phase 2 break-glass still exists on purpose.

I trust the phase split for what comes next. Phase 4 is vLLM on the cluster, Helm from
the laptop, the workload story Phase 2 deferred. That only makes sense if Phase 3 is
boring when it works: connect, run commands, disconnect, no ceremony.

Phase 3 is marked complete in the repo because the script said so — not because the agent
congratulated me, not because curl returned JSON from a bastion path I do not want as
daily life. The cluster lived on the node. I needed it on my laptop. Now it is, and the
check script is the witness.
