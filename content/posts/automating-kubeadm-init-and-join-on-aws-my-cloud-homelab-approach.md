+++
date = '2025-06-29T13:35:45-04:00'
title = 'Automating Kubeadm Init and Join on Aws My Cloud Homelab Approach'
+++

When you're setting up a Kubernetes cluster using `kubeadm`, one of the first questions is:  
**“How do I automate the init/join logic without hardcoding IPs or manually copying tokens?”**

In my [AWS-based Kubernetes homelab](https://github.com/hoaraujerome/k8s-homelab), I wanted a fully automated, reproducible setup — including both control plane and worker nodes joining the cluster automatically as soon as they boot.

This blog explains how I accomplished that using:

*   **EC2 instance tags and metadata**
    
*   **SSM Parameter Store (for secure state sharing)**
    
*   **Cloud-init & systemd (for boot-time logic)**
    

## 🧱 Background

I built a custom AMI (Ubuntu-based) using **Packer + Ansible**, used by both control plane and worker nodes. At boot, every EC2 instance checks its role and automatically does one of the following:

*   If it's the **control plane**, run `kubeadm init`, install **Cilium**, and push the join command to SSM.
   
*   If it's a **worker node**, fetch the join command from SSM and run `kubeadm join`.


This results in **zero manual steps**, even when scaling the cluster.

## 🔑 The Strategy

[Here](https://github.com/hoaraujerome/k8s-homelab/blob/main/images/config/ansible/roles/kubernetes/templates/usr/local/bin/kubeadm-init.sh.j2)'s how I approached the automation:

### 1\. **Cloud-init triggers the logic on boot**

In my AMI, I include this `cloud-init` [config](https://github.com/hoaraujerome/k8s-homelab/blob/main/images/config/ansible/roles/kubernetes/files/etc/cloud/cloud.cfg.d/99_k8s.cfg) to run a custom systemd service:

```yaml
# /etc/cloud/cloud.cfg.d/99_k8s.cfg
#cloud-config
runcmd:
  - systemctl daemon-reload
  - systemctl enable kubeadm-init.service
  - systemctl start kubeadm-init.service
```

This means the node’s role evaluation and bootstrapping start **automatically at boot**.

### 2\. **Role detection via EC2 Metadata**

Each EC2 node has a `Role` tag (`k8s-control-plane` or `k8s-worker`), and this script uses the EC2 metadata service (IMDSv2) to fetch it:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

ROLE=$(curl -s -H "X-aws-ec2-metadata-token: ${TOKEN}" \
  http://169.254.169.254/latest/meta-data/tags/instance/Role)
```

### 3\. **SSM Parameter Store for dynamic state sharing**

Since the worker node needs the control plane’s IP and the join command, I used **AWS SSM Parameter Store** to store:

*   The control plane’s **private IP**
   
*   The `kubeadm join` command, generated with `--print-join-command`
   

Example upload on the control plane:

```bash
aws ssm put-parameter \
  --name "/k8s-homelab/control-plane-private-ip" \
  --value "$CONTROL_PLANE_PRIVATE_IP" \
  --type "String" --overwrite
```

And for the join command (as a SecureString):

```bash
aws ssm put-parameter \
  --name "/k8s-homelab/worker-node-join-command" \
  --value "$JOIN_COMMAND" \
  --type "SecureString" --overwrite
```

### 4\. **Workers: Wait, Fetch, and Join**

To give the control plane time to initialize, workers wait 2 minutes, then:

*   Fetch control plane IP and add it to `/etc/hosts`
*   Retrieve the join command from SSM
*   Execute `kubeadm join`

```bash
CONTROL_PLANE_PRIVATE_IP=$(aws ssm get-parameter \
  --name "/k8s-homelab/control-plane-private-ip" \
  --query "Parameter.Value" --output text)

WORKER_NODE_JOIN_COMMAND=$(aws ssm get-parameter \
  --name "/k8s-homelab/worker-node-join-command" \
  --with-decryption --query "Parameter.Value" --output text)

eval "${WORKER_NODE_JOIN_COMMAND}"
```

## 🔄 What Could Be Improved?

*   ❌ **Avoid using never-expiring tokens**: In my current setup, the `kubeadm` join token is created with `--ttl 0`, meaning it never expires. This is fine for bootstrapping, but in a production or long-lived setup, it's a security risk. Ideally, use a short TTL and regenerate as needed via automation.
  
*   ⏳ **Replace static wait with readiness checks**: Right now, worker nodes wait a fixed 2 minutes before trying to join. A better approach would be to poll the SSM parameter or check API server health before proceeding.
  
*   📡 **Move to DNS-based discovery**: Instead of writing the control plane's IP into `/etc/hosts`, I could use private DNS or AWS Cloud Map to dynamically resolve the control plane node.
  
*   📈 **Explore scaling with Auto Scaling Groups (ASG)**: This current setup works well for static clusters, but I could extend it to support dynamic scaling by integrating with ASG and lifecycle hooks.
 
## 🎯 Final Thoughts

This was a fun and educational challenge. I used this approach to strengthen my prep for the **CKA certification**, but it’s also laying the foundation for running **production-grade workloads** on a homelab cluster I fully understand and control.

📌 Curious about the full setup? Check out the GitHub repo:  
👉 [github.com/hoaraujerome/k8s-homelab](https://github.com/hoaraujerome/k8s-homelab)

💡 Want to understand the design trade-offs and cost-saving decisions behind this setup?  
👉 Read the blog post on [design and cost decisions]({{< relref "aws-kubernetes-homelab-design-decisions.md" >}})
