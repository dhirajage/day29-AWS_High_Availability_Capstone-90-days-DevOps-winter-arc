# 🚀 Day 29/90 — DevOps Winter Arc (AWS Week Capstone)

## 📌 Project Overview

This project represents the **AWS Week Capstone** of the DevOps Winter Arc challenge. It bridges core cloud infrastructure principles—such as VPC networking, load balancing, automated scaling, database isolation, and metric monitoring—into a single production-grade, highly available architecture.

---

## 🏗️ Architecture Diagram

![AWS High Availability Architecture](./architecture/aws-ha-architecture.png)

### Visual Topology & Security Group Flow

```mermaid
graph TD
    Users([🌐 Internet Users]) -->|HTTP :80| ALB_SG

    subgraph VPC ["AWS VPC (Multi-AZ)"]
        subgraph Public_Subnets ["Public Subnets (AZ-A & AZ-B)"]
            ALB_SG["🛡️ alb-sg (Inbound: 80 from 0.0.0.0/0)"]
            ALB["⚖️ Application Load Balancer (ALB)"]
            ALB_SG --- ALB
        end

        subgraph Private_App_Subnets ["Private App Subnets (AZ-A & AZ-B)"]
            APP_SG["🛡️ app-sg (Inbound: 80 from alb-sg)"]
            ASG["🔄 Auto Scaling Group (Min: 2, Desired: 2, Max: 4)"]
            EC2_A["🖥️ EC2 Instance A (AZ-A)<br/>Nginx + Instance ID"]
            EC2_B["🖥️ EC2 Instance B (AZ-B)<br/>Nginx + Instance ID"]
            APP_SG --- ASG
            ASG --- EC2_A
            ASG --- EC2_B
        end

        subgraph Private_DB_Subnets ["Private Data Subnets"]
            DB_SG["🛡️ db-sg (Inbound: 5432 from app-sg)"]
            RDS[("🗄️ Private RDS PostgreSQL")]
            DB_SG --- RDS
        end
    end

    CW["📊 Amazon CloudWatch<br/>(CPU, 5xx Errors, UnHealthyHostCount)"]

    ALB -->|Forward / Health Check| APP_SG
    EC2_A -->|PostgreSQL :5432| DB_SG
    EC2_B -->|PostgreSQL :5432| DB_SG
    ASG -.->|Metrics & Scaling Alarms| CW
    ALB -.->|Request & Target Metrics| CW
```

### Text Flow Summary

```text
Users ➔ ALB (Public Subnets, AZ-A + AZ-B) ➔ Target Group ➔ EC2 App Instances (Private Subnets, Multi-AZ) ➔ Private RDS PostgreSQL
```

---

## 🛠️ Key Components & Implementation

1. **Networking (VPC):** Utilized a secure multi-AZ network topology featuring public subnets for the load balancer and isolated private subnets for compute and database tiers.

2. **Security Groups (Chained Least-Privilege):**
   * `alb-sg`: Accepts HTTP (`80`) traffic from the internet.
   * `app-sg`: Permits incoming traffic strictly from `alb-sg`.
   * `db-sg`: Restricts PostgreSQL (`5432`) access exclusively to `app-sg`.

3. **Launch Template & Compute:** Configured an Ubuntu instance template incorporating IMDSv2 security requirements, an SSM instance profile, and an automated Nginx bootstrap script rendering the live EC2 instance ID.

4. **Application Load Balancer & ASG:** Deployed an internet-facing ALB backed by an Auto Scaling Group (`Min: 2`, `Desired: 2`, `Max: 4`) spanning two Availability Zones with ELB health checks.

5. **Observability:** Configured CloudWatch metrics and alarms targeting host CPU utilization, 5xx errors, and target health (`UnHealthyHostCount`).

---

## 📁 Repository Structure

```text
day29-ha-app/
├── architecture/
│   └── aws-ha-architecture.png   # High-availability architecture diagram
├── scripts/                      # Nginx bootstrap scripts & user-data configuration
├── screenshots/                  # Verification & CloudWatch monitoring proofs
└── README.md                     # Project documentation & execution notes
```

---

## 🧪 Verification & Testing

* **Redundancy Test:** Sent repeated `curl` requests to the ALB DNS endpoint, confirming round-robin traffic distribution across multiple private EC2 instance IDs.
* **Failover Test:** Terminated an active worker node in the ASG to verify seamless zero-downtime auto-recovery and replacement.
* **Cleanup & Cost Control:** Performed full resource teardown (ASG, ALB, Target Group, Launch Template, CloudWatch alarms, and NAT/RDS resources) to prevent idle billing.

---

## 💡 Key Takeaways

* **Health-Check Alignment:** Target group health-check paths must map directly to an active application route (such as `/` on Nginx) to prevent false `502`/`503` errors.
* **Dependency-Aware Teardown:** Terminating EC2 instances alone does not clean up load balancers, target groups, or networking components—always verify each service after deletion.