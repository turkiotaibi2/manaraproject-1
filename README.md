# Scalable WordPress Web Application on AWS
![Alt text](./img/manara-SA-architecture.jpg)

This project showcases how to deploy a scalable and highly available WordPress-based web application architecture on AWS using EC2, Elastic Load Balancer, Auto Scaling, EFS, and Amazon RDS, CloudWatch, SNS and IAM roles.

## Project Overview
Objective: Deploy a simple web application(wordpress site) on AWS using EC2 instances, ensuring high availability and scalability with Elastic Load Balancing (ALB) and Auto Scaling Groups (ASG). The project demonstrates best practices for compute scalability, security, and cost optimization.
Architecture Type: EC2-based with external RDS and shared EFS

### Architecture Summary

AWS Services Used:

- Amazon EC2: Hosts the WordPress application
- Application Load Balancer (ALB): Routes traffic to multiple EC2 instances
- Auto Scaling Group (ASG): Maintains desired capacity and scales EC2 instances
- Amazon RDS (MySQL): Multi-AZ relational database for WordPress backend
- Amazon EFS: Shared file system mounted at /var/www/html across instances
- IAM Roles: Role-based permissions for EC2 to access CloudWatch, SSM, and other services
- CloudWatch + SNS: Performance monitoring and alerting
- Amazon VPC: Networking and security configuration

### VPC and Networking Setup

- VPC: Created a custom VPC with CIDR block 173.33.0.0/20

- Subnets:

    - 2 Public Subnets (across 2 AZs): For EC2 + ALB

    - 1 Private Subnet: For Amazon RDS (MySQL)

- Route Tables: Associated appropriate routes to subnets

- Internet Gateway: Attached to VPC for public subnets


### Amazon EFS (Shared Storage)

- Created an EFS file system in the same VPC

- Enabled mount targets in each AZ

- Mounted EFS to /var/www/html in all EC2 instances

- Ensures that all ASG-launched instances serve the same WordPress content

### Amazon RDS (Multi-AZ MySQL)

- Provisioned an RDS MySQL database in Multi-AZ mode

- Deployed in private subnet for security

- Endpoint used in WordPress wp-config.php

### EC2 Configuration

- AMI & Instance

- Amazon Linux 2 AMI

- User-data script installs Apache, PHP 8.2, MySQL client, and mounts EFS

User Data Script:

```
#!/bin/bash
yum update -y
amazon-linux-extras enable php8.2
yum clean metadata
yum install -y httpd php php-mysqlnd php-fpm php-json php-gd php-mbstring php-xml php-curl php-cli wget unzip amazon-efs-utils mysql

systemctl start httpd
systemctl enable httpd

mkdir -p /var/www/html

sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
fs-0b130b7dccab0f40e.efs.us-east-1.amazonaws.com:/ /var/www/html

chown -R apache:apache /var/www/html
sudo systemctl restart httpd
```

### Load Balancing and Auto Scaling

- Application Load Balancer (ALB) created with listener on port 80

- Target Group: Registered ASG instances using health checks

### Auto Scaling Group (ASG):

- Launch Template includes IAM role, EFS mount, and Apache config

- Configured scaling policies:

- Scale Out: Average CPU > 70%

- Scale In: Average CPU < 30%

### IAM Role for EC2

- Created IAM role: EC2WebAppRole

- Attached policies:

    - AmazonSSMManagedInstanceCore

    - CloudWatchAgentServerPolicy


- Role attached via Launch Template for all instances using ASG group

### CloudWatch & SNS Alerts

- Created SNS Topic: EC2AlertsTopic

- Subscribed email to receive alerts

- Created CloudWatch Alarm:

    - Metric: GroupAverageCPUUtilization from ASG

    - Threshold: > 70% for 5 minutes

    - Action: Send notification to EC2AlertsTopic

### WordPress Setup Notes

- WordPress files placed in EFS (/var/www/html)

- wp-config.php edited to point to RDS endpoint and DB credentials

- Apache is automatically started on boot via user data

### Security Groups Configuration
I created three dedicated security groups to ensure clear separation of concerns and secure communication between components:

- EC2 Security Group:
Allows inbound HTTP (port 80) and SSH (port 22) from the internet, and outbound NFS traffic to EFS (port 2049). This security group is also allowed by the NFS and RDS groups to enable communication.

- EFS (NFS) Security Group:
Allows inbound traffic on port 2049 (NFS) only from the EC2 security group, enabling the EC2 instances to mount the shared file system securely.

- RDS Security Group:
Restricts access to the MySQL database to only the EC2 security group on port 3306, ensuring the database is not publicly accessible and is only reachable by the application servers.

### Conclusion
This project demonstrates a scalable, fault-tolerant, load-balanced WordPress deployment using Amazon EC2, EFS, RDS, and Auto Scaling Groups. The use of Auto Scaling policies ensures that performance is optimized under varying workloads, while the shared file system guarantees consistent content delivery across all instances. The deployment also features a secure, highly available backend database, IAM role-based access control, and automated monitoring with alerting to ensure operational visibility and security.