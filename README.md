# 🏗️ AWS Project: Production-Ready VPC with Public & Private Subnets

> **Designed and implemented a secure, highly available, and scalable AWS VPC infrastructure featuring multi-AZ subnets, Application Load Balancer, Auto Scaling Group, NAT Gateways, and S3 VPC Endpoint — a production-grade architecture for real-world cloud workloads.**

---

## 📌 Overview

This project demonstrates how to build a **complete AWS network architecture** from scratch using Amazon VPC best practices. The infrastructure is designed for production workloads with a strong focus on **security**, **high availability**, and **cost-optimized traffic routing**.

![VPC Architecture Overview](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127292280.jpeg)

---

## 🧰 AWS Services Used

| Service | Purpose |
|---|---|
| **Amazon VPC** | Custom isolated network environment |
| **Public Subnets** | Host ALB, NAT Gateways, Bastion Host |
| **Private Subnets** | Host EC2 application instances |
| **Internet Gateway** | Enable internet access for public subnets |
| **NAT Gateway** | Secure outbound internet for private instances |
| **Application Load Balancer** | Distribute traffic across EC2 instances |
| **Auto Scaling Group** | Automatically scale EC2 based on demand |
| **S3 Gateway Endpoint** | Private, optimized S3 access without NAT |
| **Security Groups** | Instance-level traffic control |
| **Route Tables** | Control traffic flow per subnet |

---

## 🏛️ Architecture Design

### Network Layout

```
Region: ap-south-1 (Mumbai)
VPC CIDR: 10.0.0.0/16
│
├── Availability Zone 1 (ap-south-1a)
│   ├── Public Subnet  — 10.0.1.0/24  → NAT GW + ALB node
│   └── Private Subnet — 10.0.2.0/24  → EC2 App Servers (ASG)
│
└── Availability Zone 2 (ap-south-1b)
    ├── Public Subnet  — 10.0.3.0/24  → NAT GW + ALB node
    └── Private Subnet — 10.0.4.0/24  → EC2 App Servers (ASG)
```

### Traffic Flow

```
Internet
   │
   ▼
Internet Gateway
   │
   ▼
Application Load Balancer (Public Subnets — AZ1 & AZ2)
   │
   ├──▶ EC2 Instance (Private Subnet AZ1)
   └──▶ EC2 Instance (Private Subnet AZ2)
              │
              ▼
         NAT Gateway (for outbound internet)
              │
              ▼
         Internet Gateway
              │
              ▼ (also direct via VPC Endpoint)
         Amazon S3
```

---

## ✅ What I Implemented

### 1. 🌐 Custom VPC

- Created a **custom VPC** with CIDR `10.0.0.0/16`
- Enabled DNS hostnames and DNS resolution
- Attached an **Internet Gateway** for public internet access

![VPC and Subnet Configuration](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127302071.jpeg)

---

### 2. 🗂️ Public & Private Subnets (Multi-AZ)

- Created **2 public subnets** across 2 Availability Zones (AZ1 & AZ2)
- Created **2 private subnets** across 2 Availability Zones (AZ1 & AZ2)
- Enabled **auto-assign public IPv4** on public subnets only

#### Route Table Configuration

| Subnet Type | Destination | Target |
|---|---|---|
| Public | 0.0.0.0/0 | Internet Gateway |
| Public | 10.0.0.0/16 | local |
| Private | 0.0.0.0/0 | NAT Gateway |
| Private | 10.0.0.0/16 | local |
| Private | S3 prefix | VPC Endpoint |

---

### 3. ⚖️ Application Load Balancer (ALB)

- Deployed ALB in **both public subnets** for cross-AZ load balancing
- Created a **Target Group** pointing to EC2 instances in private subnets
- Configured **health checks** on the target group
- ALB listens on **port 80 (HTTP)** and forwards to instances on port **80**

![Application Load Balancer Setup](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127311507.jpeg)

---

### 4. 📈 Auto Scaling Group (ASG)

- Created a **Launch Template** with Amazon Linux 2023 AMI
- Configured Auto Scaling Group to deploy EC2 instances in **private subnets**
- Set scaling policies:
  - **Minimum:** 2 instances
  - **Desired:** 2 instances
  - **Maximum:** 4 instances
- Attached the ASG to the ALB Target Group

![Auto Scaling Group Configuration](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127315054.jpeg)

---

### 5. 🔒 NAT Gateways

- Deployed **1 NAT Gateway per AZ** (in public subnets) for high availability
- Allocated **Elastic IPs** for each NAT Gateway
- Updated private subnet route tables to route `0.0.0.0/0` → NAT Gateway
- Private EC2 instances can now download packages and reach the internet **without being publicly exposed**

![NAT Gateway and Routing Setup](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127325248.jpeg)

---

### 6. 📦 S3 Gateway Endpoint

- Created an **S3 Gateway VPC Endpoint** to allow private instances to access S3
- Added the endpoint route to private subnet route tables
- **Benefits:**
  - Traffic stays within the AWS network (never hits the internet)
  - No NAT Gateway charges for S3 traffic
  - Faster and more secure S3 access

```
Private EC2 ──▶ VPC Endpoint ──▶ Amazon S3
                (free, private)
                
vs.

Private EC2 ──▶ NAT Gateway ──▶ Internet ──▶ S3
                (charged per GB)
```

---

### 7. 🛡️ Security Groups

Configured layered security groups for proper traffic isolation:

| Security Group | Inbound Rules | Attached To |
|---|---|---|
| `alb-sg` | Port 80 from 0.0.0.0/0 | Application Load Balancer |
| `ec2-sg` | Port 80 from `alb-sg` only | EC2 instances (ASG) |
| `bastion-sg` | Port 22 from My IP | Bastion Host (optional) |

> ✅ **Key security principle:** EC2 instances in private subnets **only accept traffic from the ALB** — they are never directly reachable from the internet.

![Security Groups and Final Architecture](https://raw.githubusercontent.com/lalkrishna24/dsmmelk/50f243b5b2a0dc1e117ee17f2e2f86edbe97932c/1772127353977.jpeg)

---

## 📊 Infrastructure Summary

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | CIDR: 10.0.0.0/16 |
| Availability Zones | 2 | ap-south-1a, ap-south-1b |
| Public Subnets | 2 | One per AZ |
| Private Subnets | 2 | One per AZ |
| Internet Gateway | 1 | Attached to VPC |
| NAT Gateways | 2 | One per AZ (HA setup) |
| Route Tables | 3 | 1 public, 2 private (one per AZ) |
| Application Load Balancer | 1 | Cross-AZ, internet-facing |
| Target Group | 1 | Port 80, health check `/` |
| Auto Scaling Group | 1 | Min 2, Desired 2, Max 4 |
| Launch Template | 1 | Amazon Linux 2023 |
| S3 Gateway Endpoint | 1 | Private S3 access |
| Security Groups | 2 | ALB + EC2 |

---

## 🔐 Architecture Outcomes

| Goal | Result |
|---|---|
| **Security** | App servers are fully private — unreachable directly from internet ✅ |
| **High Availability** | Multi-AZ design ensures no single point of failure ✅ |
| **Scalability** | ASG automatically adds/removes EC2 instances based on load ✅ |
| **Cost Optimization** | S3 VPC Endpoint eliminates NAT charges for S3 traffic ✅ |
| **Fault Tolerance** | ALB distributes traffic; ASG replaces unhealthy instances ✅ |

---

## 💡 Key Learnings

- **Public vs Private subnets** are differentiated purely by their route table — whether `0.0.0.0/0` points to an IGW (public) or NAT GW (private)
- **NAT Gateway per AZ** is best practice — a single NAT GW creates a cross-AZ dependency and single point of failure
- **ALB + ASG integration** means the load balancer automatically receives/removes instances as the ASG scales in or out
- **S3 Gateway Endpoints are free** — unlike Interface Endpoints, they add no hourly cost and save NAT data processing charges
- **Security Group chaining** (EC2 only accepts from ALB SG) is more secure than CIDR-based rules — it's identity-based, not IP-based
- **VPC flow logs** can be enabled to monitor and troubleshoot all traffic in and out of the VPC

---

## 🔗 References

- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [ALB User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Auto Scaling User Guide](https://docs.aws.amazon.com/autoscaling/ec2/userguide/)
- [NAT Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [VPC Endpoints for S3](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints-s3.html)
- [AWS Well-Architected Framework – Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)

---

## 🏷️ Tags

`AWS` `VPC` `EC2` `ALB` `Auto Scaling` `NAT Gateway` `S3 Endpoint` `Cloud Networking` `DevOps` `Solutions Architect` `High Availability` `Multi-AZ` `Private Subnets` `Security Groups` `Production Architecture` `ap-south-1`

---

> 🌐 *All infrastructure was deployed and tested on live AWS resources in the Asia Pacific (Mumbai) region.*
