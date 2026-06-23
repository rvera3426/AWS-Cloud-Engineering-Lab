# AWS CLI Commands Reference

Complete command reference for managing AWS infrastructure from the command line. All commands used in this lab documented for quick reference.

---

## EC2 Commands

### Instance Management
```bash
# List all instances with state and IP
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# List instances with name tags
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[0].Value]' \
  --output table

# Start an instance
aws ec2 start-instances --instance-ids INSTANCE-ID

# Stop an instance
aws ec2 stop-instances --instance-ids INSTANCE-ID

# Stop multiple instances at once
aws ec2 stop-instances --instance-ids INSTANCE-ID-1 INSTANCE-ID-2

# Terminate (permanently delete) an instance
aws ec2 terminate-instances --instance-ids INSTANCE-ID

# Launch a new instance (basic)
aws ec2 run-instances \
  --image-id AMI-ID \
  --instance-type t3.micro \
  --key-name my-lab-key \
  --security-group-ids SG-ID

# Launch instance with User Data (auto-configures on boot)
aws ec2 run-instances \
  --image-id AMI-ID \
  --instance-type t3.micro \
  --key-name my-lab-key \
  --security-group-ids SG-ID \
  --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>My Server</h1>" > /var/www/html/index.html' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'

# Find latest Amazon Linux 2023 AMI in your region
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*" "Name=architecture,Values=x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

### Security Groups
```bash
# List all security groups
aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].[GroupName,GroupId]' \
  --output table

# Create a security group
aws ec2 create-security-group \
  --group-name my-sg \
  --description "My security group" \
  --vpc-id VPC-ID

# Open a port to everyone
aws ec2 authorize-security-group-ingress \
  --group-id SG-ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Open a port to your IP only
aws ec2 authorize-security-group-ingress \
  --group-id SG-ID \
  --protocol tcp \
  --port 22 \
  --cidr YOUR-IP/32

# Remove an inbound rule
aws ec2 revoke-security-group-ingress \
  --group-id SG-ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### Networking
```bash
# List all subnets with AZ and CIDR
aws ec2 describe-subnets \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' \
  --output table

# List all Elastic IPs
aws ec2 describe-addresses --output table

# List all VPCs
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,CidrBlock,IsDefault]' \
  --output table

# Allocate a new Elastic IP
aws ec2 allocate-address --domain vpc

# Associate Elastic IP with instance
aws ec2 associate-address \
  --instance-id INSTANCE-ID \
  --allocation-id ALLOCATION-ID
```

---

## S3 Commands

### Bucket Management
```bash
# List all buckets
aws s3 ls

# Create a new bucket
aws s3 mb s3://bucket-name

# Delete an empty bucket
aws s3 rb s3://bucket-name

# Enable static website hosting
aws s3 website s3://bucket-name --index-document index.html

# Get bucket website configuration
aws s3api get-bucket-website --bucket bucket-name
```

### File Operations
```bash
# Upload a file
aws s3 cp filename.txt s3://bucket-name

# Upload to specific path
aws s3 cp filename.txt s3://bucket-name/folder/filename.txt

# Download a file
aws s3 cp s3://bucket-name/filename.txt ./

# List bucket contents
aws s3 ls s3://bucket-name

# Delete a file
aws s3 rm s3://bucket-name/filename.txt

# Sync a local folder to S3
aws s3 sync ./folder s3://bucket-name

# Sync S3 to local folder
aws s3 sync s3://bucket-name ./local-folder
```

### Bucket Policies & Access
```bash
# Disable block public access
aws s3api put-public-access-block \
  --bucket bucket-name \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Add public read bucket policy
aws s3api put-bucket-policy \
  --bucket bucket-name \
  --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::bucket-name/*"}]}'

# Get bucket policy
aws s3api get-bucket-policy --bucket bucket-name

# Delete bucket policy
aws s3api delete-bucket-policy --bucket bucket-name
```

---

## Load Balancer Commands

### Application Load Balancer
```bash
# Create an Application Load Balancer
aws elbv2 create-load-balancer \
  --name my-lab-alb \
  --subnets SUBNET-ID-1 SUBNET-ID-2 SUBNET-ID-3 \
  --security-groups SG-ID \
  --scheme internet-facing \
  --type application

# List all load balancers
aws elbv2 describe-load-balancers

# Describe specific load balancer
aws elbv2 describe-load-balancers --names my-lab-alb

# Delete a load balancer
aws elbv2 delete-load-balancer --load-balancer-arn LB-ARN
```

### Target Groups
```bash
# Create a target group
aws elbv2 create-target-group \
  --name my-lab-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id VPC-ID \
  --health-check-path / \
  --health-check-interval-seconds 30

# Register instances to target group
aws elbv2 register-targets \
  --target-group-arn TARGET-GROUP-ARN \
  --targets Id=INSTANCE-ID-1 Id=INSTANCE-ID-2

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn TARGET-GROUP-ARN

# Deregister an instance from target group
aws elbv2 deregister-targets \
  --target-group-arn TARGET-GROUP-ARN \
  --targets Id=INSTANCE-ID

# Delete target group
aws elbv2 delete-target-group --target-group-arn TARGET-GROUP-ARN
```

### Listeners
```bash
# Create a listener
aws elbv2 create-listener \
  --load-balancer-arn LB-ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=TARGET-GROUP-ARN

# List listeners for a load balancer
aws elbv2 describe-listeners --load-balancer-arn LB-ARN
```

---

## IAM Commands

```bash
# List all IAM users
aws iam list-users

# List all IAM roles
aws iam list-roles

# List policies attached to a user
aws iam list-attached-user-policies --user-name USERNAME

# Create an IAM user
aws iam create-user --user-name USERNAME

# Create access key for a user
aws iam create-access-key --user-name USERNAME

# Attach a policy to a user
aws iam attach-user-policy \
  --user-name USERNAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Get current identity
aws sts get-caller-identity
```

---

## General Commands

```bash
# Check what region you are configured in
aws configure get region

# Check your current identity
aws sts get-caller-identity

# Get help for any service
aws ec2 help
aws s3 help
aws iam help
aws elbv2 help

# Get help for a specific command
aws ec2 describe-instances help
```

---

## Linux Commands On EC2

### System Information
```bash
cat /etc/os-release     # Check OS version
whoami                  # Check current user
hostname                # Check hostname
uname -a                # Check kernel version
uptime                  # Check how long server has been running
```

### Resource Monitoring
```bash
free -h                 # Check memory usage
df -h                   # Check disk space
top                     # Check running processes (q to exit)
htop                    # Better process viewer (if installed)
vmstat                  # Virtual memory stats
```

### Networking
```bash
ip addr                 # Check network interfaces and IPs
ip route                # Check routing table
netstat -tuln           # Check listening ports
curl ifconfig.me        # Check public IP from inside instance
ping google.com         # Test internet connectivity
```

### File Operations
```bash
ls -la                  # List files with permissions
cat filename            # Display file contents
nano filename           # Edit a file
sudo bash -c 'echo "text" > /path/file'  # Write file as root
chmod 400 file.pem      # Set read-only permissions on key file
```

### Service Management
```bash
sudo systemctl start SERVICE    # Start a service
sudo systemctl stop SERVICE     # Stop a service
sudo systemctl restart SERVICE  # Restart a service
sudo systemctl enable SERVICE   # Enable service on boot
sudo systemctl disable SERVICE  # Disable service on boot
sudo systemctl status SERVICE   # Check service status
```

### Package Management (Amazon Linux)
```bash
sudo dnf update -y              # Update all packages
sudo dnf install PACKAGE -y     # Install a package
sudo dnf remove PACKAGE -y      # Remove a package
sudo dnf search PACKAGE         # Search for a package
```

---

## Windows PowerShell Commands

### SSH Connection
```powershell
# Navigate to key file location
cd "C:\AWS LAB"

# Connect to EC2 instance
ssh -i "my-lab-key.pem" ec2-user@YOUR-PUBLIC-IP

# Fix .pem file permissions (run if SSH refuses the key)
takeown /F "C:\AWS LAB\my-lab-key.pem"
icacls "C:\AWS LAB\my-lab-key.pem" /inheritance:r /remove "BUILTIN\Users" /remove "Everyone" /remove "BUILTIN\Administrators"

# Find .pem file if you forgot where it is
Get-ChildItem -Path C:\ -Recurse -Filter "*.pem" 2>$null
```

---

## EC2 State Codes

| Code | State |
|---|---|
| 0 | Pending |
| 16 | Running |
| 32 | Shutting down |
| 48 | Terminated |
| 64 | Stopping |
| 80 | Stopped |

---

## Common ARN Formats

```
EC2 Instance:    arn:aws:ec2:REGION:ACCOUNT:instance/INSTANCE-ID
S3 Bucket:       arn:aws:s3:::BUCKET-NAME
S3 Object:       arn:aws:s3:::BUCKET-NAME/OBJECT-KEY
IAM User:        arn:aws:iam::ACCOUNT:user/USERNAME
IAM Role:        arn:aws:iam::ACCOUNT:role/ROLE-NAME
Lambda:          arn:aws:lambda:REGION:ACCOUNT:function:FUNCTION-NAME
Load Balancer:   arn:aws:elasticloadbalancing:REGION:ACCOUNT:loadbalancer/app/NAME/ID
Target Group:    arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/NAME/ID
```

---

## Free Tier Limits

| Service | Free Tier |
|---|---|
| EC2 t3.micro | 750 hours/month for 12 months |
| S3 | 5GB storage, 20K GET, 2K PUT for 12 months |
| Lambda | 1M requests/month forever |
| CloudWatch | 10 metrics, 10 alarms, 1M API requests for 12 months |
| CloudShell | Always free |
| Elastic IP | Free when attached to running instance |
| Data Transfer | 100GB outbound/month for 12 months |

---

*Last updated: June 22, 2026*
