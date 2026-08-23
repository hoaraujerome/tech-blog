+++
date = '2026-08-23T12:58:22-04:00'
title = 'Done When the Script Says So, Not When the Agent Says So'
+++

I recently finished [Phase 2](https://github.com/hoaraujerome/vllm-deployment/blob/main/phase2/README.md) of a personal project: a single-node Kubernetes cluster on AWS,
built greenfield with Terraform, Packer, Ansible, and kubeadm. Most of the typing happened
in Cursor, with an agent proposing diffs and me approving or rejecting them.

Several times the conversation reached a tone I now recognize as dangerous: *"This should
work."* *"Phase 2 is essentially complete."* *"You can mark this done."*

I did not mark it done. I ran the check script instead.

That gap — between the agent's confidence and my definition of finished — is what this
post is about. Not how to bootstrap a cluster. The repo holds that. This is about how I
decided **when** the work was actually finished while building with an agent.

## The agent is fast; it is not accountable

Agentic coding feels like having a very eager junior who never sleeps. It will refactor
three modules because you mentioned naming consistency. It will copy patterns from a
reference repo you cited once, including the parts you did not want. It will summarize
progress in complete sentences that sound like a stand-up update.

None of that is malice. It is optimization for *plausible continuation* — keep the
conversation moving, keep generating patches.

Accountability still sits with the engineer. In my day job that is obvious. On a personal
project, with nobody reviewing my merge requests, it is easier to let the agent's narrative
become my narrative. *We got the smoke test working* can quietly become *ship it* even
when half the gates have never run end-to-end.

I did not want that. So I borrowed a framing from Andrew Ng's
["loop engineering"](https://www.deeplearning.ai/the-batch/issue-359/) newsletter
and made it operational before the Terraform grew teeth.

## Three loops, three different jobs

In [The Batch, Issue 359](https://www.deeplearning.ai/the-batch/issue-359/), Andrew Ng
describes three nested feedback loops for building products — from fast agentic iteration
to slower human steering to external reality. The diagram is his; my labels below are how
I applied it to a solo infra project.

[![Three key product development loops: Agentic Coding Loop (~minutes), Developer Feedback Loop (~hours), and External Feedback Loop (~days)](https://charonhub.deeplearning.ai/content/images/2026/06/3KeyDevelopmentLoops_v4a-1.jpg)](https://www.deeplearning.ai/the-batch/issue-359/)

*Figure: "3 key product development loops" from
[The Batch, Issue 359](https://www.deeplearning.ai/the-batch/issue-359/) by Andrew Ng /
DeepLearning.AI. Image hosted at deeplearning.ai; click to open the source article.*

Ng names them **agentic coding**, **developer feedback**, and **external feedback**. I
mapped them onto Phase 2 like this:

**Agentic coding loop (~minutes).** Edit, run checks, paste failures back, repeat. The
agent drives throughput; I drive direction. In Ng's terms: coding agent ↔ product spec
and evals. My evals were the gate script.

**Developer feedback loop (~hours).** I reject bad architecture, update written
constraints, restart the agentic loop. Bootstrap belongs in the AMI first boot, not in a
laptop-side Ansible playbook — even when the agent kept offering the playbook path because
it looked familiar. Spec and vision live in the README reject list.

**External feedback loop (~days).** Metrics, users, cost surprises. Phase 2 deferred almost
all of that. One private node, no vLLM yet, no ingress. The cluster factory had to work
before the workload story mattered. My preview of this loop was cruder: reboot the instance
from the AWS console and see if the cluster still passed checks.

The mistake I wanted to avoid was collapsing the agentic loop and the developer feedback
loop. Letting the agent close the latter because the former *felt* green.

## Objective checks before module sprawl

Early in Phase 2 I wrote a validation ladder: numbered gates, skip flags, presets for
daily use versus full greenfield runs. I wrote it **before** the infrastructure layout
hardened — on purpose.

The rule was simple. The agent does not declare Phase 2 done. A script exiting zero does.

That sounds rigid. It is. It is also the only way I found to keep agent sessions from
forking into alternate realities. In one session the EICE tunnel works; in another the
Packer variables were never wired across process boundaries; in a third kubelet was
enabled in the script but nobody had rebooted the node. Each session had its own partial
truth.

Gates turned partial truth into a single boolean.

I am not claiming this replaces code review or good design docs. For a solo builder using
an agent, it replaced the missing second pair of eyes with something worse than a human
but better than vibes: **repeatable falsification**.

## What the developer feedback loop actually looked like

The developer feedback loop was not abstract. It was me saying no in writing and making
the agent restart.

No bastion — EC2 Instance Connect Endpoint, because that is what I trusted from an
earlier homelab. No fork of that homelab repo — greenfield paths, inspirational only. No
vLLM baked into the AMI. No merging Terraform state between the Packer builder network
and the cluster network. No nullable variables "for flexibility." A long reject list, frozen
in the README, so the agentic loop had guardrails.

When the agent violated a constraint, I did not argue in chat. I pointed at the constraint
and re-ran the ladder. Chat is where confidence inflates. The ladder is where lies die.

## The moment it felt real

The full ladder going green was satisfying. It was also insufficient emotionally.

What moved Phase 2 from *probably works* to *I trust this* was smaller: I rebooted the
EC2 instance from the AWS console — not from a script I had tuned — and ran the daily
preset again. Node Ready. Smoke pod up. Kubelet still enabled for boot.

That was an external-feedback preview on a toy scale. Not users, but physics. Power
cycle does not care about our conversation history.

Until then I had been optimizing for the agent's definition of progress: the next gate,
the next fix, the next green run. The reboot was my developer-feedback verdict on top of
the agentic loop's automation. **Finished** is not **merged**. Finished is **survives
contact with reality you did not script**.

## What I believe now

If you build infrastructure with an agent in 2026, treat the agent as an accelerator, not
an arbiter. Write down what "done" means before you write the modules. Make "done" a
command anyone — including future you, tired on a Sunday — can run without interpreting
chat logs.

Pre-commit hooks and formatters are hygiene. They are not the contract. The contract is
the integration ladder: plan, apply, bake, bootstrap, schedule, smoke — whatever your
phase actually requires, end to end.

I will keep blogging the opinions and the surprises. The step-by-step belongs in
[the repo](https://github.com/hoaraujerome/vllm-deployment) and in whatever LLM you pair
with next. Phase 2 is marked complete there because the script said so — after a reboot,
not because the agent congratulated me.

Phase 3 is WireGuard and laptop-side kubectl. I already know the agent will have opinions.
I already know which script will decide when **that** phase is done.
