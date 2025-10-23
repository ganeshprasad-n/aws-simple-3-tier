## 🚀 AWS Simple 3-Tier Architecture

This project demonstrates a secure, scalable, and modular 3-tier architecture.

It includes:
- **Custom VPC** with public, private, and isolated subnets  
- **EC2 instances** with custom AMIs and Auto Scaling  
- **RDS (MySQL)** setup with subnet groups and IAM authentication  
- **Application Load Balancers** with Target Groups  
- **Multiple EC2 access methods**: SSM, ICE, Bastion  
- **Cost optimization and security best practices**

📌 Designed for hands-on learning of 3-tier architecture. All infrastructure is provisioned using AWS Console manually.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 🌐 VPC Setup

This section outlines the foundational networking layer for the 3-tier architecture using AWS VPC, subnets, route tables, and security groups.

---

### 🧱 Basic VPC Elements

1. **Create VPC**  
   - Enable `Public DNS hostname` setting

2. **Create Internet Gateway (IGW)**  
   - Attach it to the VPC

3. **Create Subnets in 2 AZs**  
   - For each AZ: `public`, `web`, `app`, `DB` (or just `public`, `web`, `DB`)

4. **Enable Auto-Assign Public IP**  
   - For the two public subnets

5. **Create Route Tables**  
   - Public RT → for public subnets  
   - Private RT → for web, app, DB subnets  
   - NAT RT (optional) → only if using NAT EC2 instance  
   > ⚠️ A subnet can be associated with only one route table.  
   > For multi-AZ NAT, create separate private RTs per AZ and attach respective NAT instances.

6. **Associate Subnets to Route Tables**  
   - Link each subnet to its respective route table

7. **Add IGW Route to Public RT**  
   - Destination: `0.0.0.0/0`  
   - Target: Internet Gateway

8. **Create NAT Gateway**  
   - Must be launched in a public subnet

9. **Add NAT Route to Private RT**  
   - Destination: `0.0.0.0/0`  
   - Target: NAT Gateway

---

### 🔐 Security Groups

Create the following security groups with least privilege access:

- **Bastion / Jump Server**  
  - Allow SSH (port 22) from your IP

- **ALB Frontend**  
  - Allow HTTP (port 80) and HTTPS (port 443) from `0.0.0.0/0`

- **ALB Backend**  
  - Same as frontend

- **Instance Connect (Outbound)**  
  - Allow outbound SSH (port 22) to VPC CIDR  
  > 🔒 No inbound rule required

- **SSH-SG**  
  - Allow SSH (port 22) from your IP  
  - Allow SSH from Instance Connect SG

- **Web Frontend**  
  - Allow HTTP/HTTPS from ALB Frontend SG  
  - Allow SSH from Bastion SG and Instance Connect SG

- **App Backend**  
  - Allow HTTP/HTTPS from ALB Backend SG  
  - Allow SSH from Bastion SG and Instance Connect SG

- **DB-RDS**  
  - Allow MySQL (port 3306) from App Backend SG

- **EFS**  
  - Allow NFS (port 2049) from Web and App SGs  
  - Allow SSH (port 22) from Web and App SGs  
  - After creation, add:  
    - Allow NFS (port 2049) from EFS SG itself

- **NAT Instance** *(optional)*  
  - Only if using EC2-based NAT

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 🗄️ Database Setup (Amazon RDS - Free Tier)

This section outlines the creation of a MySQL RDS instance using the AWS Free Tier. The setup ensures secure connectivity, IAM-based authentication, and cost-conscious configuration.

---

### 🛠️ RDS Configuration Steps

1. **Choose Database Creation Method**  
   - Select `Standard create`

2. **Engine Options**  
   - Engine type: `MySQL`  
   - Engine version: as per requirement  
   - Remaining options: leave default

3. **Template**  
   - Select `Free tier`

4. **Availability & Durability**  
   - Leave default settings

5. **Settings**  
   - DB instance identifier: `your-db-name`  
   - Master username: `admin`  
   - Credentials management: `Self-managed`  
   - Auto-generate password: **Uncheck**  
   - Master password: `admin#54321`

6. **Instance Configuration**  
   - DB instance class: `Burstable (t-class)`  
   - CPU & Memory: select based on need  
   - Filters: leave default

7. **Storage**  
   - Size: `20GB`  
   - Auto-scaling: **Uncheck**

8. **Connectivity**  
   - Compute resource: `Don’t connect to EC2`  
   - Network type: `IPv4`  
   - VPC: select your custom VPC  
   - DB subnet group: select your own group  
   - Public access: `No`  
   - VPC security group: select existing DB SG  
   - Availability Zone: `a` or `b`  
   - RDS Proxy: **Uncheck**  
   - Certificate authority: leave default  
   - Database port: `3306`

9. **Tags** *(optional)*  
   - Add tags for environment or owner tracking

10. **Database Authentication**  
    - Enable `Password and IAM database authentication`

11. **Monitoring**  
    - Leave default settings

12. **Additional Configuration**  
    - Database options: leave default  
    - Backup: **Uncheck** `Enable automated backups` and `Enable encryption`  
    - Maintenance:  
      - **Uncheck** `Enable auto minor version upgrade`  
      - Maintenance window: leave default  
    - Enable deletion protection: **Check** (optional)


