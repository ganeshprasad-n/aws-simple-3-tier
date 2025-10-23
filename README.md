# AWS 3-Tier Architecture Implementation

A hands-on project implementing a scalable 3-tier architecture on AWS using VPC, EC2, Auto Scaling, Load Balancers, and RDS MySQL.

## Overview

This project demonstrates the setup of a production-grade 3-tier architecture entirely through the AWS Console. The implementation focuses on understanding core AWS networking concepts, high availability patterns, and infrastructure best practices.

**What's included:**
- Custom VPC with public and private subnet segmentation
- Custom AMIs for Web and Application tiers
- Auto Scaling Groups for dynamic scaling
- Application Load Balancers for traffic distribution
- RDS MySQL database with proper isolation
- Security groups implementing least privilege access

**Tech Stack:** AWS (VPC, EC2, ALB, ASG, RDS, NAT Gateway)

---

## Architecture Design

### Network Layout

```
VPC CIDR: 10.0.0.0/16

Public Subnets (ALB placement):
├── 10.0.1.0/24 (us-east-1a) - public-subnet-1a
└── 10.0.2.0/24 (us-east-1b) - public-subnet-2b

Private Subnets - Web Tier:
├── 10.0.3.0/24 (us-east-1a) - web-subnet-3a
└── 10.0.4.0/24 (us-east-1b) - web-subnet-4b

Private Subnets - Application Tier:
├── 10.0.5.0/24 (us-east-1a) - app-subnet-5a
└── 10.0.6.0/24 (us-east-1b) - app-subnet-6b

Private Subnets - Database Tier:
├── 10.0.7.0/24 (us-east-1a) - db-subnet-7a
└── 10.0.8.0/24 (us-east-1b) - db-subnet-8b
```

### Traffic Flow

```
Internet → IGW → Frontend ALB (Public Subnets)
    ↓
Web Servers (Private Subnets: 10.0.3.0/24, 10.0.4.0/24)
    ↓
Backend ALB (Public Subnets)
    ↓
App Servers (Private Subnets: 10.0.5.0/24, 10.0.6.0/24)
    ↓
RDS MySQL (Private Subnets: 10.0.7.0/24, 10.0.8.0/24)
```

**Note:** This is an infrastructure learning project. Application logic and database integration are not implemented.

---

## Implementation Steps

### 1. VPC Configuration

**Create VPC:**
```
Name: 3tier-vpc
CIDR: 10.0.0.0/16
Enable DNS hostnames: Yes
Enable DNS resolution: Yes
```

**Internet Gateway:**
- Create and attach IGW to VPC

**Subnets:**

Create 8 subnets across 2 availability zones:

| Subnet Name | CIDR | AZ | Type |
|-------------|------|-----|------|
| public-subnet-1a | 10.0.1.0/24 | us-east-1a | Public |
| public-subnet-2b | 10.0.2.0/24 | us-east-1b | Public |
| web-subnet-3a | 10.0.3.0/24 | us-east-1a | Private |
| web-subnet-4b | 10.0.4.0/24 | us-east-1b | Private |
| app-subnet-5a | 10.0.5.0/24 | us-east-1a | Private |
| app-subnet-6b | 10.0.6.0/24 | us-east-1b | Private |
| db-subnet-7a | 10.0.7.0/24 | us-east-1a | Private |
| db-subnet-8b | 10.0.8.0/24 | us-east-1b | Private |

Enable auto-assign public IPv4 for both public subnets.

**Route Tables:**

Public Route Table:
```
Routes:
- 10.0.0.0/16 → local
- 0.0.0.0/0 → Internet Gateway

Associated Subnets:
- public-subnet-1a
- public-subnet-2b
```

Private Route Table:
```
Routes:
- 10.0.0.0/16 → local
- 0.0.0.0/0 → NAT Gateway

Associated Subnets:
- All 6 private subnets
```

**NAT Gateway:**
- Create in public-subnet-1a
- Allocate Elastic IP
- Update private route table

---

### 2. Security Groups Configuration

Create the following security groups with proper inbound/outbound rules:

#### Jump-server-SG (Bastion Host)
```yaml
Name: Jump-server-SG
Description: Bastion host for SSH access

Inbound Rules:
  - Type: SSH, Port: 22, Source: Your IP address
```

#### ALB-frontend (Frontend Load Balancer)
```yaml
Name: ALB-frontend
Description: Frontend ALB security group

Inbound Rules:
  - Type: HTTP, Port: 80, Source: 0.0.0.0/0
  - Type: HTTPS, Port: 443, Source: 0.0.0.0/0
```

#### ALB-backend (Backend Load Balancer)
```yaml
Name: ALB-backend
Description: Backend ALB security group

Inbound Rules:
  - Type: HTTP, Port: 80, Source: 0.0.0.0/0
  - Type: HTTPS, Port: 443, Source: 0.0.0.0/0
```

#### instance-connect-SG (AWS Instance Connect)
```yaml
Name: instance-connect-SG
Description: Instance Connect endpoint

Inbound Rules: None

Outbound Rules:
  - Type: SSH, Port: 22, Destination: VPC CIDR (10.0.0.0/16)
```

#### SSH-SG (General SSH Access)
```yaml
Name: SSH-SG
Description: SSH access control

Inbound Rules:
  - Type: SSH, Port: 22, Source: Your IP address
  - Type: SSH, Port: 22, Source: instance-connect-SG
```

#### web-frontend-SG (Web Tier Instances)
```yaml
Name: web-frontend-SG
Description: Web tier security group

Inbound Rules:
  - Type: HTTP, Port: 80, Source: ALB-frontend
  - Type: HTTPS, Port: 443, Source: ALB-frontend
  - Type: SSH, Port: 22, Source: Jump-server-SG
  - Type: SSH, Port: 22, Source: instance-connect-SG
```

#### app-backend-SG (Application Tier Instances)
```yaml
Name: app-backend-SG
Description: Application tier security group

Inbound Rules:
  - Type: HTTP, Port: 80, Source: ALB-backend
  - Type: HTTPS, Port: 443, Source: ALB-backend
  - Type: SSH, Port: 22, Source: Jump-server-SG
  - Type: SSH, Port: 22, Source: instance-connect-SG
```

#### db-SG (Database)
```yaml
Name: db-SG
Description: RDS MySQL security group

Inbound Rules:
  - Type: MySQL/Aurora, Port: 3306, Source: app-backend-SG
```

#### EFS-SG (Elastic File System)
```yaml
Name: EFS-SG
Description: Shared file system security group

Inbound Rules:
  - Type: NFS, Port: 2049, Source: web-frontend-SG
  - Type: NFS, Port: 2049, Source: app-backend-SG
  - Type: SSH, Port: 22, Source: web-frontend-SG
  - Type: SSH, Port: 22, Source: app-backend-SG
  - Type: NFS, Port: 2049, Source: EFS-SG (add after creation)
```

> **Note:** The EFS-SG self-referencing rule must be added after the security group is created.

---

### 3. AMI Creation

Launch 2 temporary EC2 instances in the public subnet for initial configuration:

**Instance 1 - Web Server:**

```bash
# Connect to instance
ssh -i your-key.pem ec2-user@<public-ip>

# Install Apache
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

# Configure web content
cd /var/www/html
sudo nano index.html
```

Add the following content:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Web Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>This is my Web Server</h1>
        <p>Frontend Tier</p>
    </div>
</body>
</html>
```

Test: `curl http://localhost`

**Instance 2 - App Server:**

Repeat the above steps but use this content:
```html
<!DOCTYPE html>
<html>
<head>
    <title>App Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
            color: white;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>This is my App Server</h1>
        <p>Application Tier</p>
    </div>
</body>
</html>
```

**Create AMIs:**

For each instance:
1. EC2 Console → Select instance
2. Actions → Image and templates → Create image
3. Name: `web-server-ami-v1` or `app-server-ami-v1`
4. Add appropriate tags

Wait for AMI status to become "Available", then terminate both temporary instances.

---

### 4. RDS MySQL Setup

**Create DB Subnet Group:**
```
Name: db-subnet-group
VPC: 3tier-vpc
Subnets: db-subnet-7a, db-subnet-8b
```

**Create RDS Instance:**
```
Engine: MySQL (latest version)
Template: Free tier
DB Instance Identifier: 3tier-mysql-db
Master Username: admin
Master Password: admin#54321

Instance Class: db.t3.micro
Storage: 20 GB General Purpose SSD
Storage Autoscaling: Disabled

Connectivity:
- VPC: 3tier-vpc
- Subnet Group: db-subnet-group
- Public Access: No
- Security Group: db-sg
- Port: 3306

Authentication: Password + IAM authentication

Backup: Disabled (for cost optimization)
Encryption: Disabled (for cost optimization)
Auto minor version upgrade: Disabled
```

---

### 5. Target Groups

**Web Target Group:**
```
Name: web-target-group
Protocol: HTTP
Port: 80
VPC: 3tier-vpc

Health Checks:
- Protocol: HTTP
- Path: /
- Interval: 30 seconds
- Timeout: 5 seconds
- Healthy threshold: 2
- Unhealthy threshold: 2
```

**App Target Group:**
```
Name: app-target-group
Protocol: HTTP
Port: 80
VPC: 3tier-vpc

Health Checks: (same as above)
```

Do not register any targets manually - Auto Scaling Groups will handle this.

---

### 6. Load Balancers

**Frontend ALB:**
```
Name: frontend-alb
Scheme: internet-facing
IP type: IPv4
VPC: 3tier-vpc

Network Mapping:
- us-east-1a: public-subnet-1a
- us-east-1b: public-subnet-2b

Security Group: frontend-alb-sg

Listener: HTTP:80 → Forward to web-target-group
```

**Backend ALB:**
```
Name: backend-alb
Scheme: internet-facing
IP type: IPv4
VPC: 3tier-vpc

Network Mapping:
- us-east-1a: public-subnet-1a
- us-east-1b: public-subnet-1b

Security Group: backend-alb-sg

Listener: HTTP:80 → Forward to app-target-group
```

---

### 7. Launch Templates

**Web Server Launch Template:**
```
Name: web-server-launch-template
AMI: web-server-ami-v1
Instance Type: t2.micro
Security Groups: web-server-sg

Resource Tags:
- Name: web-server
- Tier: Frontend
```

**App Server Launch Template:**
```
Name: app-server-launch-template
AMI: app-server-ami-v1
Instance Type: t2.micro
Security Groups: app-server-sg

Resource Tags:
- Name: app-server
- Tier: Backend
```

---

### 8. Auto Scaling Groups

**Web Tier ASG:**
```
Name: web-asg
Launch Template: web-server-launch-template

Network:
- VPC: 3tier-vpc
- Subnets: web-subnet-3a, web-subnet-4b

Load Balancer: Attach to web-target-group
Health Check: ELB
Grace Period: 300 seconds

Capacity:
- Desired: 2
- Minimum: 2
- Maximum: 4
```

**App Tier ASG:**
```
Name: app-asg
Launch Template: app-server-launch-template

Network:
- VPC: 3tier-vpc
- Subnets: app-subnet-5a, app-subnet-6b

Load Balancer: Attach to app-target-group
Health Check: ELB
Grace Period: 300 seconds

Capacity:
- Desired: 2
- Minimum: 2
- Maximum: 4
```

---

## Testing

**Frontend Access:**
```
http://frontend-alb-XXXXXXXXXX.us-east-1.elb.amazonaws.com
```
Should display: "This is my Web Server"

**Backend Access:**
```
http://backend-alb-XXXXXXXXXX.us-east-1.elb.amazonaws.com
```
Should display: "This is my App Server"

**Verify:**
- Check target groups show 2 healthy targets each
- Refresh ALB URLs multiple times to verify load balancing
- Verify instances are running in private subnets

---

## Key Learnings

**Networking:**
- VPC design with proper CIDR planning
- Public vs private subnet usage and routing
- NAT Gateway for outbound internet access
- Security group layering for defense in depth

**Compute:**
- AMI creation for reusable configurations
- Launch Templates for consistent deployments
- Auto Scaling for high availability
- Health checks and automatic recovery

**Load Balancing:**
- ALB configuration and target group integration
- Health check implementation
- Traffic distribution across AZs

**Database:**
- RDS setup with subnet groups
- Database isolation in private subnets
- IAM authentication configuration

---

## Cost Considerations

Free tier eligible resources:
- t2.micro EC2 instances (750 hours/month)
- db.t3.micro RDS (750 hours/month)
- ALB (750 hours/month)

Potential costs:
- NAT Gateway (~$0.045/hour + data transfer)
- EBS snapshots for AMIs
- Data transfer beyond free tier limits

**Recommendation:** Set up billing alerts and monitor usage regularly.

---

## Cleanup

To avoid charges, delete resources in this order:

1. Set ASG desired capacity to 0, then delete ASGs
2. Delete both Load Balancers
3. Delete Target Groups
4. Delete Launch Templates
5. Deregister and delete AMIs (plus associated snapshots)
6. Delete RDS instance and DB subnet group
7. Delete NAT Gateway and release Elastic IP
8. Delete subnets, route tables, Internet Gateway
9. Delete Security Groups
10. Delete VPC

---

## Troubleshooting

**503 Error from ALB:**
- Check target health in target groups
- Verify security groups allow traffic from ALB
- Ensure Apache is running on instances

**Instances not launching:**
- Review ASG activity history for errors
- Verify AMI is available in the region
- Check service quotas for vCPU limits

**Cannot access private instances:**
- This is expected - instances are in private subnets
- Access through ALB DNS name or set up bastion host

---

## Technical Skills Demonstrated

- AWS VPC networking and subnet architecture
- Security group configuration and network isolation
- EC2 instance management and AMI creation
- Auto Scaling and high availability implementation
- Application Load Balancer configuration
- RDS database setup and connectivity
- Linux administration (Apache web server)
- Infrastructure documentation

---

This project showcases practical AWS infrastructure skills applicable to real-world scenarios. All resources were provisioned manually through the AWS Console to ensure thorough understanding of each component and their interactions.