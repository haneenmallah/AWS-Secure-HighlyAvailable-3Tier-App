# AWS-Secure-HighlyAvailable-3Tier-App
# AWS Cloud Capstone Project: 3-Tier Highly Available & Secure Architecture

[![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Architecture](https://img.shields.io/badge/Architecture-3--Tier%20VPC-orange?style=for-the-badge)]()
[![Security](https://img.shields.io/badge/Security-Full%20Isolation-green?style=for-the-badge)]()

**Author:** Haneen Mallah  
**Target Environment:** AWS `us-east-1`  
**Architecture:** 3-Tier Enterprise Web Application (ALB ➔ EC2 Auto Scaling Group ➔ Isolated RDS MySQL)  
**Security Model:** Full Network Isolation, Zero Static Credentials, Defense-in-Depth  

---

## System Architecture Overview

This repository documents the complete design, deployment, and verification of a production-grade 3-Tier cloud application architecture built from scratch on Amazon Web Services (AWS).

### Key Architectural Pillars:
* **High Availability:** Application and Database tiers are redundantly deployed across two Availability Zones (`us-east-1a` and `us-east-1b`).
* **Complete Network Isolation:** EC2 application servers and Amazon RDS MySQL reside in private subnets with strictly **no public IPv4 addresses assigned**. Public internet ingress is restricted exclusively to the Application Load Balancer (`capstone-2026`).
* **Secure Internal Routing:** AWS service communications (S3 for config storage, SSM for management) are routed internally via **Gateway and Interface VPC Endpoints**. Outbound internet traffic for package deployment (`httpd`, `mariadb`) is strictly one-way through a single **NAT Gateway**.
* **Zero Static Credentials:** Compute instances rely entirely on **IAM Roles (`Capstone-EC2-Role`)** with temporary credentials supplied by IMDSv2. No SSH keys or access keys are used or stored.
* **Automated Elasticity:** A dynamic **Auto Scaling Group (`Capstone-ASG`)** automatically provisions and terminates EC2 instances based on real-time CPU utilization metrics

---

## Network Infrastructure & Subnet Layout

| Subnet Name | Type | Availability Zone | CIDR Block | Assigned Resources |
| :--- | :--- | :--- | :--- | :--- |
| `Capstone-Public-A` | Public | `us-east-1a` | `10.0.1.0/24` | Ingress ALB Node, NAT Gateway |
| `Capstone-Public-B` | Public | `us-east-1b` | `10.0.2.0/24` | Ingress ALB Node |
| `Capstone-App-A` | Private | `us-east-1a` | `10.0.11.0/24` | EC2 App Instance, SSM Endpoints |
| `Capstone-App-B` | Private | `us-east-1b` | `10.0.12.0/24` | EC2 App Instance, SSM Endpoints |
| `Capstone-DB-A` | Private | `us-east-1a` | `10.0.21.0/24` | Primary RDS Instance |
| `Capstone-DB-B` | Private | `us-east-1b` | `10.0.22.0/24` | Multi-AZ Secondary RDS Instance |

> **Engineering Rationale (NAT Gateway vs. VPC Endpoints):**  
> Private EC2 instances require outbound internet access to install Linux packages via `dnf`, but must remain completely isolated from incoming internet connections. A NAT Gateway provides strictly one-way egress. To optimize security and data transfer costs, internal service calls (S3 & Systems Manager) bypass the NAT Gateway entirely and route through internal VPC Endpoints

---

## Security Groups Chaining & Defense-in-Depth


Security Groups are configured as a tightly chained defense perimeter where each tier explicitly references the Security Group ID of the preceding tier rather than IP ranges:

| Security Group Name | Traffic Type | Allowed Port | Inbound Source Condition |
| :--- | :--- | :--- | :--- |
| **`Capstone-ALB-SG`** | HTTP | Port `80` | Public Internet (`0.0.0.0/0`) |
| **`Capstone-EC2-SG`** | HTTP | Port `80` | `Capstone-ALB-SG` ID Only |
| **`Capstone-RDS-SG`** | MySQL/Aurora | Port `3306` | `Capstone-EC2-SG` ID Only |
| **`Capstone-Endpoint-SG`** | HTTPS | Port `443` | `Capstone-EC2-SG` ID Only |

* **Stateless Network ACLs (`Capstone-App-NACL`):** Configured with explicit Inbound Rule 120 and Outbound Rule 130 permitting ephemeral ports (`1024-65535`) to handle session response traffic.
* **Port 22 Removal:** SSH is completely disabled across all tiers. Remote administration is managed exclusively via **AWS Systems Manager Session Manager** over HTTPS.

---

## User Data Initialization Script

```bash
#!/bin/bash
BUCKET="capstone-2026"
DBHOST="capstone-db.cgfy8q4m8zzw.us-east-1.rds.amazonaws.com"

dnf install -y httpd mariadb105
systemctl enable --now httpd

TOKEN=$(curl -s -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
M="[http://169.254.169.254/latest/meta-data](http://169.254.169.254/latest/meta-data)"
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" $M/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" $M/placement/availability-zone)
LIP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" $M/local-ipv4)
REGION=${AZ%?}

if aws s3 cp "s3://$BUCKET/application-config.txt" /tmp/config.txt --region "$REGION" >/dev/null 2>&1; then
    S3STATUS="OK"; S3CLASS="ok"
else
    S3STATUS="FAILED"; S3CLASS="bad"
fi

if timeout 3 bash -c "cat < /dev/null > /dev/tcp/$DBHOST/3306" 2>/dev/null; then
    DBSTATUS="OK"; DBCLASS="ok"
else
    DBSTATUS="FAILED"; DBCLASS="bad"
fi

echo "OK" > /var/www/html/health.html

cat > /var/www/html/index.html <<EOF <!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8">
<title>AWS Cloud Capstone</title><style>
body{background:#050e16;color:#e8f4f8;font-family:sans-serif;display:grid;place-items:center;height:100vh;margin:0}
.card{background:#0c1e2c;border:1px solid #193a50;border-radius:16px;padding:32px 40px}
h1{margin:0 0 4px;font-size:21px}
h2{margin:0 0 20px;font-size:13px;color:#00d4c8;font-weight:400}
p{margin:7px 0;font-family:monospace;font-size:14px}
.ok{color:#3ddc97} .bad{color:#ff6b6b}
</style></head><body><div class="card">
<h1>AWS Cloud Capstone</h1>
<h2>3-Tier Highly Available Architecture - Owner: haneen</h2>
<p>Instance ID: $IID</p>
<p>Availability Zone: $AZ</p>
<p>Private IP: $LIP</p>
<p>S3 Connectivity: <span class="$S3CLASS">$S3STATUS</span></p>
<p>Database Connectivity: <span class="$DBCLASS">$DBSTATUS</span></p>
</div></body></html>
EOF
Comprehensive Verification & Proof of Execution
Application Ingress & Multi-AZ Load Balancing: Verified via ALB DNS URL (capstone-alb-259804992.us-east-1.elb.amazonaws.com) Traffic successfully balanced across us-east-1a and us-east-1b instances
Database Isolation Test: Direct connections to the RDS endpoint from CloudShell/External networks timed out (timeout 5), confirming zero public exposure

IAM Role & IMDSv2 Inspection: Executed aws sts get-caller-identity and ls -la ~/.aws via SSM. Confirmed no static credentials stored on disk

Auto Scaling Under Stress: CPU stress script (for i in $(seq 1 $(nproc))...) triggered Capstone-CPU-High CloudWatch Alarm, automatically provisioning a 3rd healthy instance

Scale-In Recovery: Terminating stress processes triggered Capstone-CPU-Low Alarm, safely decommissioning the extra instance back to the minimum capacity of 2



## Complete Documentation
The full engineering report in PDF format is available for direct download:
*  **[Download Complete PDF Report](./AWS_Capstone_Project_Report_Haneen_Mallah.pdf)**
