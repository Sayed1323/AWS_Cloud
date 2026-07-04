# AWS_Cloud
AWS Prod infrastructure for handling Applications 
 
________________________________________
STEP-BY-STEP IMPLEMENTATION GUIDE
STEP 1: Create the VPC
AWS Console > VPC > Create VPC

VPC name: production-vpc
IPv4 CIDR block: 10.0.0.0/16
IPv6 CIDR block: (Leave blank)
Tenancy: Default
DNS hostnames: Enable
DNS resolution: Enable

Click: Create VPC
________________________________________
STEP 2: Create Internet Gateway (IGW)
AWS Console > VPC > Internet Gateways > Create Internet Gateway

Name: production-igw

Click: Create Internet Gateway

Then: Attach to VPC
  - Select: production-vpc
  - Click: Attach Internet Gateway
Purpose: Allows internet traffic to enter and exit the VPC
________________________________________
STEP 3: Create Public Subnet
AWS Console > VPC > Subnets > Create Subnet

VPC ID: production-vpc
Subnet name: public-subnet-az1
Availability Zone: us-east-1a (choose your AZ)
IPv4 CIDR block: 10.0.1.0/24
Enable auto-assign public IPv4 address: YES ✓

Click: Create Subnet
________________________________________
STEP 4: Create Private Subnets (App & Database)
For Application Tier:
VPC ID: production-vpc
Subnet name: private-app-subnet-az1
Availability Zone: us-east-1a
IPv4 CIDR block: 10.0.2.0/24
Enable auto-assign public IPv4 address: NO

Click: Create Subnet
For Database Tier:
VPC ID: production-vpc
Subnet name: private-db-subnet-az2
Availability Zone: us-east-1b (Different AZ for Multi-AZ)
IPv4 CIDR block: 10.0.3.0/24
Enable auto-assign public IPv4 address: NO

Click: Create Subnet
________________________________________
STEP 5: Create Route Tables
PUBLIC Route Table:
AWS Console > VPC > Route Tables > Create Route Table

Name: public-rt
VPC: production-vpc

Edit Routes:
  Destination: 0.0.0.0/0
  Target: Internet Gateway (production-igw)
  
Edit Subnet Associations:
  - Add: public-subnet-az1
PRIVATE Route Table (for App Servers):
Name: private-app-rt
VPC: production-vpc

Edit Routes:
  Destination: 0.0.0.0/0
  Target: NAT Gateway (we'll create this next)
  
Edit Subnet Associations:
  - Add: private-app-subnet-az1
PRIVATE Route Table (for Database):
Name: private-db-rt
VPC: production-vpc

(NO ROUTES - DB doesn't need outbound internet)

Edit Subnet Associations:
  - Add: private-db-subnet-az2
________________________________________
STEP 6: Create NAT Gateway (For Outbound Traffic from Private Subnet)
AWS Console > EC2 > Elastic IPs > Allocate Elastic IP Address

Allocate this first, then:

AWS Console > VPC > NAT Gateways > Create NAT Gateway

Name: nat-gateway-az1
Subnet: public-subnet-az1  ⚠️ NAT goes in PUBLIC subnet
Elastic IP allocation ID: (select the one you just created)

Click: Create NAT Gateway

Wait for it to show "Available" status
Then update the private-app-rt route to point to this NAT Gateway
________________________________________
STEP 7: Create Security Groups
Security Group 1: Public (Load Balancer)
AWS Console > EC2 > Security Groups > Create Security Group

Name: sg-public-alb
Description: Allow HTTP/HTTPS from internet
VPC: production-vpc

Inbound Rules:
  ✓ HTTP    | Port 80   | Source: 0.0.0.0/0 (Anywhere)
  ✓ HTTPS   | Port 443  | Source: 0.0.0.0/0 (Anywhere)

Outbound Rules:
  ✓ All traffic | Destination: 0.0.0.0/0 (All)

Click: Create Security Group
Purpose: ✅ Allows end users from internet to access Load Balancer
________________________________________
Security Group 2: Private (Application Servers)
Name: sg-private-app
Description: Allow traffic from Load Balancer only
VPC: production-vpc

Inbound Rules:
  ✓ Custom TCP | Port 8080  | Source: sg-public-alb (Select from dropdown)
  ✓ SSH        | Port 22    | Source: YOUR_IP (For maintenance)

Outbound Rules:
  ✓ All traffic | Destination: 0.0.0.0/0 (To DB & outbound)

Click: Create Security Group
Purpose: ✅ Allows traffic ONLY from Load Balancer to App Servers
________________________________________
Security Group 3: Private (Database)
Name: sg-private-db
Description: Allow database traffic from app servers only
VPC: production-vpc

Inbound Rules:
  ✓ MySQL/Aurora | Port 3306 | Source: sg-private-app (Select from dropdown)
  OR
  ✓ PostgreSQL   | Port 5432 | Source: sg-private-app

  (Choose based on your database type)

Outbound Rules:
  ✗ (No outbound needed for DB)

Click: Create Security Group
Purpose: ✅ Allows traffic ONLY from App Servers to Database. Isolates DB completely.
________________________________________
STEP 8: Create Load Balancer (ALB)
AWS Console > EC2 > Load Balancers > Create Load Balancer

Choose: Application Load Balancer

Configuration:
  Name: production-alb
  Scheme: Internet-facing
  IP address type: IPv4

Network Mapping:
  VPC: production-vpc
  Subnets: 
    ✓ public-subnet-az1
    ✓ public-subnet-az1b (Add second AZ for high availability)

Security Groups:
  ✓ sg-public-alb

Listeners:
  HTTP:80 → Forward to target group (we'll create)
  HTTPS:443 → (Create certificate & set up later)

Create Target Group:
  Name: app-tg
  Protocol: HTTP
  Port: 8080
  VPC: production-vpc
  Health check path: /health
  
  Register targets: (Add EC2 instances from private subnet)
________________________________________
STEP 9: Create EC2 Instances (Application Servers)
AWS Console > EC2 > Instances > Launch Instances

Configuration:
  AMI: Amazon Linux 2 (or Ubuntu)
  Instance type: t3.medium (or appropriate size)
  VPC: production-vpc
  Subnet: private-app-subnet-az1
  Security group: sg-private-app
  
  Create 2 instances minimum (for high availability)

Note: Instances in private subnet won't have public IPs,
but they can access internet via NAT Gateway
________________________________________
STEP 10: Create RDS Database
AWS Console > RDS > Create Database

Database creation method: Standard create
Engine: MySQL 8.0 / PostgreSQL / Aurora (pick your choice)

DB instance identifier: production-db
Master username: admin
Master password: (Create strong password)

DB instance class: db.t3.micro (or larger)
Storage: 20 GB (adjust as needed)

VPC: production-vpc
DB subnet group: Create new
  - Select: private-db-subnet-az1 and private-db-subnet-az2
Publicly accessible: NO ✓
VPC security group: sg-private-db

Multi-AZ: Enable ✓ (for production)
Backup retention: 30 days ✓
Enable encryption: Yes ✓

Click: Create Database
________________________________________
SUMMARY: Traffic Flow
🌐 INTERNET (End Users)
    ↓ Port 80/443
    
🔓 INTERNET GATEWAY (IGW)
    ↓
    
📊 LOAD BALANCER (sg-public-alb)
    ↓ Port 8080
    
🚀 APPLICATION SERVERS in PRIVATE SUBNET (sg-private-app)
    ↓ Port 3306/5432
    
🗄️ DATABASE in PRIVATE SUBNET (sg-private-db)
    ← ISOLATED - Only accepts from App tier
________________________________________
Security Best Practices Applied:
Layer	Security	How
Internet Entry	Only Port 80/443 allowed	IGW + ALB security group
App to Public	Not directly exposed	Private subnet + NAT
App to Database	Only specific port	DB security group restricts to app tier
Database Isolation	Completely private	No public IP, only app tier access
Outbound from App	Controlled	NAT Gateway masks source IP
Database Backup	Automated	RDS multi-AZ + S3 snapshots
________________________________________
Quick Troubleshooting:
❌ Apps can't reach database?
•	Check: sg-private-db allows inbound from sg-private-app on port 3306
❌ Apps can't download packages from internet?
•	Check: NAT Gateway exists, route table points to it, NAT has Elastic IP
❌ Users can't reach website?
•	Check: IGW is attached, Load Balancer has public IP, sg-public-alb allows 80/443
Would you like me to create a detailed Terraform/CloudFormation script to automate this entire setup? 🚀



