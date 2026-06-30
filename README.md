# AWS Auto Scaling with Application Load Balancer

A hands-on AWS infrastructure project demonstrating automatic EC2 scaling based on CPU load, with traffic distributed across instances via an Application Load Balancer.

---

## Overview

This project sets up a production-style auto-scaling environment on AWS. A custom AMI is used as the base for a Launch Template, which powers an Auto Scaling Group that scales EC2 instances in and out automatically based on CPU utilization. An Application Load Balancer distributes traffic evenly across all healthy instances across two Availability Zones.

---

## Architecture

```
Internet
   │
   ▼
Application Load Balancer (LB1)  —  Internet-facing, HTTP:80
   │
   ▼
Target Group (TG1)  —  HTTP:80, Instance type
   │
   ├── EC2 Instance 1 (t3.micro | ap-south-1a)
   ├── EC2 Instance 2 (t3.micro | ap-south-1a)
   ├── EC2 Instance 3 (t3.micro | ap-south-1b)  ← auto scaled out
   └── EC2 Instance 4 (t3.micro | ap-south-1b)  ← auto scaled out
         ▲
Auto Scaling Group (ASG1)
 Min: 2 | Desired: 4 | Max: 5
 Policy: Target Tracking — CPU > 70%
         ▲
Launch Template (LT1)
 AMI: server1-backup | Instance: t3.micro | Key: server
```

---

## Tech Stack

| Component | Detail |
|---|---|
| AWS EC2 | t3.micro — Amazon Linux |
| Custom AMI | server1-backup — pre-configured with Apache + custom index.html |
| Launch Template | LT1 — defines instance config for the ASG |
| Auto Scaling Group | ASG1 — Min 2 / Desired 4 / Max 5 |
| Scaling Policy | Target Tracking — Average CPU Utilization > 70% |
| Application Load Balancer | LB1 — Internet-facing, HTTP:80, 2 AZs |
| Target Group | TG1 — HTTP:80, Instance type, ELB health checks |
| stress | CPU load simulator to trigger scale-out events |

---

## Setup & Configuration

### Step 1 — Custom AMI

A base EC2 instance (`server1`) was configured with Apache and a custom status page, then saved as a custom AMI (`server1-backup`) to use as a golden image for all instances launched by the ASG.

![Custom AMI](screenshots/01-custom-ami.png)

---

### Step 2 — Launch Template

Launch Template `LT1` was created using the custom AMI. It defines the instance blueprint: AMI ID, key pair (`server`), and security group — so every instance launched by the ASG is identical and pre-configured.

![Launch Template](screenshots/02-launch-template.png)

---

### Step 3 — Instance Type

The primary instance type is set to `t3.micro` (2 vCPU, 1 GiB Memory), selected manually in the ASG configuration.

![Instance Type](screenshots/03-instance-type.png)

---

### Step 4 — Auto Scaling Group

`ASG1` was created using `LT1`. The group is at desired capacity with all 4 instances healthy.

![ASG Overview](screenshots/04-asg-overview.png)

---

### Step 5 — Group Size & Scaling Limits

Group size is configured with Desired: 4, Min: 2, Max: 5. The ASG will never go below 2 instances and will scale up to 5 under load.

![ASG Group Size](screenshots/05-asg-group-size.png)

---

### Step 6 — Network & Availability Zones

The ASG is spread across two subnets in `ap-south-1a` and `ap-south-1b` with **Balanced best effort** AZ distribution — if one zone fails, instances launch in the other.

![Network AZ Config](screenshots/06-network-az.png)

---

### Step 7 — Load Balancer Attached to ASG

Target Group `TG1` (HTTP, attached to `LB1`) is registered with the ASG so every new instance automatically joins the load balancer.

![LB Attached to ASG](screenshots/07-lb-attached-asg.png)

---

### Step 8 — Health Check Configuration

ELB health checks are enabled on the ASG with a 300-second grace period — giving new instances time to boot and initialize before health checks begin.

![Health Check Config](screenshots/08-health-check-config.png)

---

### Step 9 — Application Load Balancer

`LB1` is Active, Internet-facing, application type, spanning 2 Availability Zones. The metrics panel shows live request count and response time.

![Load Balancer Active](screenshots/09-load-balancer-active.png)

---

### Step 10 — Load Balancer Details & DNS

LB1 details showing the public DNS name used to access the application: `LB1-238032626.ap-south-1.elb.amazonaws.com`

![LB Details DNS](screenshots/10-lb-details-dns.png)

---

### Step 11 — Listeners & Rules

LB1 listens on HTTP:80 and forwards 100% of traffic to Target Group `TG1`.

![LB Listeners Rules](screenshots/11-lb-listeners-rules.png)

---

### Step 12 — Network Mapping

LB1 is mapped across both AZ subnets (`ap-south-1a` and `ap-south-1b`) ensuring traffic reaches instances in either zone.

![LB Network Mapping](screenshots/12-lb-network-mapping.png)

---

### Step 13 — Target Group Health

`TG1` shows **5/5 healthy targets** on HTTP:80 — all instances passing health checks with 0 unhealthy.

![Target Group Health](screenshots/13-target-group-health.png)

---

## Scaling Test

The `stress` tool was used to push CPU above the 70% threshold and trigger the Target Tracking policy.

```bash
# Install stress on Amazon Linux
sudo amazon-linux-extras install epel -y
sudo yum install stress -y

# Push CPU to trigger scale-out
stress --cpu 2 --timeout 300
```

### Scale-Out & Scale-In Activity

The activity log shows the complete scaling cycle:

- **Scale-out**: `AlarmHigh` fired → ASG launched new instances (capacity increased)
- **Scale-in**: `AlarmLow` fired → ASG terminated instances step by step (4→3→2) back toward minimum

![Scaling Activity](screenshots/14-scaling-activity.png)

---

### EC2 Instances During Scaling

The instances list shows the result of the scaling cycle — 5 instances total, 4 running across both AZs, 1 terminated as part of scale-in. All running instances show **3/3 status checks passed**.

![EC2 Instances](screenshots/15-ec2-instances.png)

---

## Result — Live via ALB DNS

The custom status page served through the Application Load Balancer DNS name, confirming end-to-end traffic flow from the internet through the ALB to the EC2 instances.

![Browser via ALB](screenshots/16-browser-alb.png)

---

## Key Learnings

- Building a custom AMI as a golden image for consistent instance provisioning
- Using Launch Templates to define repeatable instance configurations for ASGs
- Configuring target tracking scaling policies to automate scale-out and scale-in
- Spreading ASG instances across multiple AZs for fault tolerance
- Attaching ELB health checks to ASG so unhealthy instances are automatically replaced
- Using `stress` to simulate real CPU load and validate the full scaling pipeline

---

## Repository Structure

```
.
├── README.md
├── screenshots/
│   ├── 01-custom-ami.png
│   ├── 02-launch-template.png
│   ├── 03-instance-type.png
│   ├── 04-asg-overview.png
│   ├── 05-asg-group-size.png
│   ├── 06-network-az.png
│   ├── 07-lb-attached-asg.png
│   ├── 08-health-check-config.png
│   ├── 09-load-balancer-active.png
│   ├── 10-lb-details-dns.png
│   ├── 11-lb-listeners-rules.png
│   ├── 12-lb-network-mapping.png
│   ├── 13-target-group-health.png
│   ├── 14-scaling-activity.png
│   ├── 15-ec2-instances.png
│   └── 16-browser-alb.png
└── web/
    └── index.html
```

---

## Author

Madan Babu (Madmax) — Pune, India  
[GitHub](https://github.com/madansharma619-cmyk)
