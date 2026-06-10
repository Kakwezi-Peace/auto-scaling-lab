# Auto Scaling Lab — AWS EC2 Auto Scaling + ALB

## What This Builds

```
Internet Users
      |
  [ALB - one public DNS endpoint]
      |             |
[EC2 Instance 1]  [EC2 Instance 2]   ← auto-scaled, private
   (AZ-1)           (AZ-2)
      |             |
  [NAT Gateway] ← lets private instances download packages
```

---

## Deployment Steps

### Step 1 — Create a GitHub Repository

1. Go to [github.com](https://github.com) and sign in
2. Click **New repository**
3. Name it: `auto-scaling-lab`
4. Set it to **Public**
5. Click **Create repository**
6. Upload both files (`template.yaml` and `README.md`) to the repo

---

### Step 2 — Deploy via AWS CloudFormation Console

> If GitSync is not required, you can deploy directly from the console.

1. Open [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation)
2. Click **Create stack → With new resources (standard)**
3. Choose **Upload a template file**
4. Upload `template.yaml`
5. Click **Next**
6. Stack name: `AutoScalingLab`
7. Leave parameters as default (AMI ID is auto-fetched)
8. Click **Next → Next → Submit**
9. Wait ~5–8 minutes for the stack to reach `CREATE_COMPLETE`

---

### Step 3 — (Optional) Deploy via CloudFormation GitSync

GitSync connects your GitHub repo so CloudFormation auto-deploys when you push changes.

1. In CloudFormation → Click **Create stack → With new resources**
2. Choose **Sync from Git**
3. Connect your GitHub account when prompted
4. Select your `auto-scaling-lab` repository
5. Set the template path to: `template.yaml`
6. Complete stack creation

---

### Step 4 — Get the ALB DNS Name

After the stack finishes:
1. In CloudFormation → Click your stack → **Outputs** tab
2. Copy the value next to `ALBDNS` — it looks like:
   ```
   http://AutoScalingLabALB-xxxxxxxx.us-east-1.elb.amazonaws.com
   ```
3. Open it in your browser — you'll see a page with the **Instance ID**, **IP**, and **AZ**
4. **Refresh the page** multiple times — you may see different instance IDs (load balancing!)

---

## Live Demo Steps

### Demo 1: Load Balancing

1. Open the ALB DNS URL in your browser
2. Keep refreshing — note the Instance ID changes between requests
3. This proves the ALB is distributing traffic across multiple servers

### Demo 2: Scale-Out via CPU Stress Test

You need to connect to an EC2 instance using **AWS Systems Manager (SSM)** — no SSH needed.

**Step-by-step:**

1. Open [EC2 Console](https://console.aws.amazon.com/ec2) → **Instances**
2. Click on a running instance → **Connect** → **Session Manager** → **Connect**
3. A browser terminal opens. Run this command to stress the CPU:

```bash
stress --cpu 4 --timeout 300 &
```

This maxes out the CPU for 5 minutes, simulating heavy load.

4. Go to **EC2 → Auto Scaling Groups → AutoScalingLabASG → Activity** tab
5. Watch new instances being added automatically within 1–3 minutes
6. Go back to the ALB URL and refresh — you'll see new Instance IDs appear

**To stop the stress test early:**
```bash
pkill stress
```

### Demo 3: Scale-In (Automatic)

After the stress test ends, CPU returns to normal. Within a few minutes, the ASG automatically **removes** the extra instances it added (scale-in). You'll see the instance count drop back down.

---

## Scaling Policy Explained Simply

| Setting | Value | Meaning |
|---|---|---|
| Target CPU | 30% | If avg CPU > 30%, add servers |
| Min instances | 1 | Never go below 1 server |
| Desired | 1 | Start with 1 server |
| Max instances | 4 | Never create more than 4 servers |
| Scale-out cooldown | 60s | Wait 60s before adding another instance |
| Scale-in cooldown | 120s | Wait 2 min before removing an instance |

---

## Architecture Summary

| Component | Purpose |
|---|---|
| VPC (10.0.0.0/16) | Private network on AWS |
| Public Subnets (AZ1, AZ2) | Where the ALB lives |
| Private Subnets (AZ1, AZ2) | Where the EC2 instances live (no direct internet) |
| Internet Gateway | Connects the VPC to the internet |
| NAT Gateway | Lets private EC2s download packages |
| ALB | Single public endpoint; distributes traffic |
| Target Group | List of EC2s the ALB knows about |
| Launch Template | Blueprint for new EC2 instances |
| Auto Scaling Group | Manages how many EC2s run |
| Scaling Policy | CPU > 30% → add instances |
| SSM Role | Lets you connect to EC2 without SSH |

---

## Security Best Practices Applied

- EC2 instances are in **private subnets** — not reachable from internet directly
- **No SSH key** on instances — access via SSM Session Manager only
- EC2 security group only allows traffic **from the ALB**, nothing else
- ALB security group only allows **HTTP (port 80)** from the internet
