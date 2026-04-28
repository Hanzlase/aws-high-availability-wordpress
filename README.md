<div align="center">

<img src="https://cdn.simpleicons.org/amazonaws/FF9900" width="72" alt="AWS Logo" />

# Highly Available WordPress on AWS

### Production-grade, fault-tolerant, and auto-scaling WordPress infrastructure

[![AWS](https://img.shields.io/badge/Amazon_AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=FF9900)](https://aws.amazon.com/)
[![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)](https://wordpress.org/)
[![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)](https://httpd.apache.org/)
[![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Amazon Linux](https://img.shields.io/badge/Amazon_Linux_2023-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/linux/)

</div>

---

## <img src="https://cdn.simpleicons.org/readme/018EF5" width="22" alt="README" /> Overview

This repository documents the deployment of a **production-grade, highly available, and scalable WordPress infrastructure** on Amazon Web Services (AWS). The project transitions from a monolithic *single point of failure* setup to a decoupled, multi-tier architecture capable of handling traffic spikes and maintaining **99.9% uptime**.

---

## <img src="https://cdn.simpleicons.org/diagramsdotnet/F08705" width="22" alt="Architecture" /> Architecture Overview

> The infrastructure is built within a custom **VPC** spread across multiple **Availability Zones (AZs)** to ensure fault tolerance.

```
Internet Users
      │
      ▼
┌─────────────────────┐
│   Amazon CloudFront │  ← Global CDN / Edge Caching
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Application Load   │  ← Single Entry Point
│  Balancer  (ALB)    │
└────────┬────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐      ← Auto Scaling Group (ASG)
│ EC2   │ │ EC2   │        Min: 2  |  Max: 3
│ (AZ1) │ │ (AZ2) │
└───┬───┘ └───┬───┘
    └────┬────┘
         │
         ▼
┌─────────────────────┐
│   Amazon RDS        │  ← Primary (Write)
│   (MySQL Master)    │
└──────────┬──────────┘
           │  Replicates
           ▼
┌─────────────────────┐
│   RDS Read Replica  │  ← Offloads Read Traffic
└─────────────────────┘
```

### Key Components

| Icon | Service | Role |
|------|---------|------|
| <img src="https://cdn.simpleicons.org/amazonaws/FF9900" width="18" /> | **Amazon CloudFront** | Global CDN — edge caching & reduced latency |
| <img src="https://cdn.simpleicons.org/awselasticloadbalancing/8C4FFF" width="18" /> | **Application Load Balancer (ALB)** | Distributes traffic across healthy EC2 instances |
| <img src="https://cdn.simpleicons.org/amazonec2/FF9900" width="18" /> | **Auto Scaling Group (ASG)** | Dynamically manages EC2 fleet via Golden AMI |
| <img src="https://cdn.simpleicons.org/amazonrds/527FFF" width="18" /> | **Amazon RDS (MySQL)** | Primary DB + Read Replica for resilience |
| <img src="https://cdn.simpleicons.org/amazons3/569A31" width="18" /> | **Amazon S3** | Static asset storage |
| <img src="https://cdn.simpleicons.org/amazonaws/FF9900" width="18" /> | **Security Groups** | Layered ALB → ASG → RDS least-privilege model |

---

## <img src="https://cdn.simpleicons.org/rocket/4A90D9" width="22" alt="Phases" /> Implementation Phases

### <img src="https://cdn.simpleicons.org/amazonec2/FF9900" width="18" /> Phase 0 — Baseline Infrastructure

The initial environment consists of a primary EC2 instance running the **LAMP stack** (Linux, Apache, MySQL, PHP) connected to a managed Amazon RDS instance.

- Configured dedicated Security Groups for the web server and the database.
- Verified database connectivity and completed the initial WordPress installation.

---

### <img src="https://cdn.simpleicons.org/amazonec2/FF9900" width="18" /> Phase 1 — The "Golden Image" (AMI)

To support rapid scaling, a custom **Amazon Machine Image (AMI)** was created from the fully configured web server. This image contains:

- Operating system & kernel configuration
- Web server (Apache) settings
- Application code & WordPress configuration

The AMI serves as a **master template** for all instances launched by the Auto Scaling Group.

---

### <img src="https://cdn.simpleicons.org/awselasticloadbalancing/8C4FFF" width="18" /> Phase 2 — Traffic Distribution (ALB)

An **internet-facing Application Load Balancer** was deployed as the single entry point for users.

| Setting | Value |
|---------|-------|
| Target Group | `alb-tg` — routes traffic to the web tier |
| Health Checks | Automatically stops routing to unhealthy instances |
| Scheme | Internet-facing |

---

### <img src="https://cdn.simpleicons.org/awsautoscaling/FF9900" width="18" /> Phase 3 — High Availability & Auto Scaling

The **Auto Scaling Group (ASG)** ensures the application handles varying loads elastically.

| Parameter | Value |
|-----------|-------|
| Launch Template AMI | Custom Golden Image |
| Instance Type | `t2.micro` |
| Security Group | `asg-sg` |
| Minimum Instances | **2** |
| Maximum Instances | **3** |
| Availability Zones | 2 AZs |

**Bootstrap (User Data) script applied on every instance launch:**

```bash
#!/bin/bash
yum update -y
sudo service httpd restart
```

---

### <img src="https://cdn.simpleicons.org/amazonaws/FF9900" width="18" /> Phase 4 — Global Acceleration (CloudFront)

A **CloudFront distribution** was integrated to serve as the edge/CDN layer.

| Path Behaviour | Cache Policy |
|----------------|-------------|
| `/wp-admin/*` | Custom behaviour — bypass cache |
| `/wp-content/*` | Long-lived cache for static assets |
| `/wp-includes/*` | Long-lived cache for WordPress core assets |

- **HTTPS Redirect** enforced — all HTTP requests are permanently redirected to HTTPS.

---

### <img src="https://cdn.simpleicons.org/amazonrds/527FFF" width="18" /> Phase 5 — Database Fault Tolerance (RDS Read Replica)

An **RDS Read Replica** was created to split the database workload:

- Read queries are offloaded from the primary master, improving performance during high-traffic periods.
- Provides an additional layer of **data redundancy**.
- In case of primary failure, the replica can be promoted to master.

---

## <img src="https://cdn.simpleicons.org/stackshare/0690FA" width="22" alt="Tech Stack" /> Tech Stack

<div align="center">

[![AWS EC2](https://img.shields.io/badge/EC2-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/ec2/)
[![AWS RDS](https://img.shields.io/badge/RDS-527FFF?style=for-the-badge&logo=amazon-rds&logoColor=white)](https://aws.amazon.com/rds/)
[![AWS ALB](https://img.shields.io/badge/ALB-8C4FFF?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/elasticloadbalancing/)
[![AWS CloudFront](https://img.shields.io/badge/CloudFront-FF9900?style=for-the-badge&logo=amazon-cloudfront&logoColor=white)](https://aws.amazon.com/cloudfront/)
[![AWS ASG](https://img.shields.io/badge/Auto_Scaling-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/autoscaling/)
[![AWS S3](https://img.shields.io/badge/S3-569A31?style=for-the-badge&logo=amazon-s3&logoColor=white)](https://aws.amazon.com/s3/)

[![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)](https://wordpress.org/)
[![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)](https://httpd.apache.org/)
[![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://www.mysql.com/)
[![PHP](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://www.php.net/)
[![Amazon Linux](https://img.shields.io/badge/Amazon_Linux_2023-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=FF9900)](https://aws.amazon.com/linux/)

</div>

---

## <img src="https://cdn.simpleicons.org/googleanalytics/E37400" width="22" alt="Results" /> Results

| <img src="https://cdn.simpleicons.org/statuspage/05C16E" width="16" /> Achievement | Description |
|------------|-------------|
| **Zero Downtime** | If one AZ fails, the ASG automatically launches healthy instances in the remaining AZ |
| **Auto Scalability** | The system grows and shrinks based on real-time demand, optimising costs |
| **Layered Security** | Database is isolated in a private tier — accessible only by the web servers |
| **Global Performance** | CloudFront edge nodes serve cached content with minimal latency worldwide |
| **Data Redundancy** | RDS Read Replica ensures reads are available even during primary maintenance |

---

## <img src="https://cdn.simpleicons.org/github/181717" width="22" alt="Author" /> Author

<div align="center">

**Muhammad Hanzla**
*Software Engineering Student · Cloud & DevOps Enthusiast*

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Hanzlase)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/hanzlase/)
[![AWS](https://img.shields.io/badge/Cloud_Practitioner-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/certification/certified-cloud-practitioner/)

</div>

---

<div align="center">

<sub>Built with <img src="https://cdn.simpleicons.org/amazonaws/FF9900" width="12" /> AWS · Documented for learning & portfolio purposes</sub>

</div>