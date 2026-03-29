+++
date = '2025-06-29T14:32:38-04:00'
title = 'From Goals to Constraints to Costs: Designing a Lean AWS Kubernetes Homelab'
+++

## 🧭 Why Build a Homelab?

I recently completed the first phase of my **cloud-native homelab** — a Kubernetes cluster on AWS built from scratch with `kubeadm`, provisioned using **Terraform**, **Packer**, **Ansible**, and **Cilium**.

This wasn't just for fun (though it was). I designed this homelab as:

*   A hands-on way to **prepare for the CKA certification**
    
*   A platform to **host real-world workloads** later
    
*   A personal sandbox to **understand what’s happening under the hood**, not just run `kubectl apply`
    

## 🔒 My Hard Constraints (Non-Negotiable by Design)

Before talking about cost, it’s important to share the **non-cost-related constraints** that framed the entire project:

*   ✅ **No managed Kubernetes** like EKS — I wanted to use kubeadm to learn how Kubernetes is really bootstrapped and managed.
    
*   ✅ **Use AWS** — my current job is 100% Azure, so I took this as an opportunity to get some **multi-cloud fluency**.
    
*   ✅ **1 control plane and 1 worker node** — I wanted to simulate a real-world cluster but still keep things minimal and reproducible.
    

All of these decisions were **intentional learning constraints** — I wasn’t optimizing for cost *yet*, but for **knowledge depth**.

## 💰 Cost-Conscious Design Decisions

Once I set the learning and platform boundaries, it was time to think about **keeping the monthly AWS bill reasonable**. Below are four specific design choices that helped me achieve that.

### 1\. 🖼️ One Shared AMI for Both Node Types

Instead of creating two separate AMIs — one for the control plane and one for worker nodes — I built a **single AMI** using Packer + Ansible that works for both.

✅ **Pros:**

*   Halves the EBS snapshot storage cost
    
*   Reduces image maintenance overhead
    

⚠️ **Trade-off:**

*   Requires boot-time logic to determine node role and run `kubeadm init` or `kubeadm join` accordingly.
    

### 2\. 🧠 Small EC2 Instances That Meet Kubeadm's Minimum Requirements

I opted for **small EC2 instance types** (e.g., t3.small) that just fulfill kubeadm's minimum memory/CPU requirements.

✅ **Pros:**

*   Low hourly cost
    
*   Sufficient for learning, testing, and small workloads
    

⚠️ **Trade-off:**

*   Limited resources may make the cluster sluggish under real load
    
*   No HA or autoscaling, but that wasn’t the goal for this phase
    

💭 **Why not spot instances or ephemeral workers?**  
I briefly considered using temporary EC2s for worker nodes to reduce cost further. But since this was my first time **bootstrapping a cluster end-to-end automatically**, I wanted to keep things simple and reproducible before introducing complexity like auto-scaling groups or lifecycle hooks.

### 3\. 🔐 EC2 Instance Connect over Bastion, Session Manager, or VPN

To SSH into nodes (which are in private subnets), I chose **EC2 Instance Connect** instead of:

*   Maintaining a **bastion host** 24/7
    
*   Using **AWS Session Manager**
    

✅ **Pros:**

*   No extra instance running all the time
    
*   Works out of the box for Amazon Linux and Ubuntu
    
*   Easy to use in a pinch via the AWS console
    

⚠️ **Trade-off:**

*   Not as flexible as a fully managed SSH jumpbox
    
*   Can be slower to use for multi-hop scripting or advanced setups
    

### 4\. 🔑 Sharing the Kubeadm Join Command via Parameter Store

Control plane generates the `kubeadm join` command and pushes it to **AWS SSM Parameter Store** (as a SecureString). At boot, the worker node fetches it securely and joins the cluster.

✅ **Pros:**

*   Simple to implement
    
*   Works well with cloud-init and early boot workflows
    
*   Inexpensive at low read/write volumes
    

## 📝 Key Takeaways

Designing this Kubernetes homelab involved much more than clicking “Launch” on EC2. It was about balancing:

*   **Learning goals** (like CKA prep and kubeadm hands-on)
    
*   **Platform constraints** (AWS, no managed K8s, 2-node limit)
    
*   **Cost-saving choices** that didn’t compromise the intent
    

I hope this breakdown helps you **think critically about your own homelab architecture** — whether for learning, cert prep, or even a production-like setup.

In the next phase, I plan to evolve this cluster toward real-world workloads and maybe explore cost-aware auto-scaling patterns or monitoring stacks.

 ➡️ **Curious about how I automated the** `kubeadm init` and `join` logic at boot using EC2 metadata, cloud-init, and SSM Parameter Store?  
Check out the companion post: 👉 [Automating kubeadm Init & Join on AWS: My Cloud Homelab Approach]({{< relref "automating-kubeadm-init-and-join-on-aws-my-cloud-homelab-approach.md" >}})

You can also explore the entire project on Github: 👉 [**github.com/hoaraujerome/k8s-homelab**](https://github.com/hoaraujerome/k8s-homelab)

Until then, happy tinkering!
