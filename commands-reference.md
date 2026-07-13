# AWS CLI Commands Reference

Complete command reference for managing AWS infrastructure from the command line, covering every service used across all seven lab sessions.

---

## EC2 Commands

### Instance Management
```bash
# List all instances with state and IP
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table

# List instances with name tags
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[0].Value]' --output table

# Start / stop instances
aws ec2 start-instances --instance-ids INSTANCE-ID
aws ec2 stop-instances --instance-ids INSTANCE-ID-1 INSTANCE-ID-2

# Terminate (permanently delete)
aws ec2 terminate-instances --instance-ids INSTANCE-ID

# Enable detailed monitoring (1-minute metrics instead of 5)
aws ec2 monitor-instances --instance-ids INSTANCE-ID-1 INSTANCE-ID-2

# Launch an instance with User Data (auto-configures at boot)
aws ec2 run-instances \
  --image-id AMI-ID --instance-type t3.micro --key-name my-lab-key \
  --security-group-ids SG-ID \
  --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'

# Find the latest Amazon Linux 2023 AMI in your region
aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*" "Name=architecture,Values=x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text
```

### Security Groups
```bash
aws ec2 describe-security-groups --query 'SecurityGroups[*].[GroupName,GroupId]' --output table
aws ec2 create-security-group --group-name my-sg --description "My SG" --vpc-id VPC-ID
aws ec2 authorize-security-group-ingress --group-id SG-ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 revoke-security-group-ingress --group-id SG-ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### Networking, Subnets, and Route Tables
```bash
# List subnets with AZ and CIDR
aws ec2 describe-subnets --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' --output table

# Create a new subnet
aws ec2 create-subnet --vpc-id VPC-ID --cidr-block 172.31.48.0/20 --availability-zone us-east-2a

# Inspect route tables (this determines public vs private)
aws ec2 describe-route-tables --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]'

# Create a route table with no internet route (= private)
aws ec2 create-route-table --vpc-id VPC-ID

# Associate a route table with a subnet
aws ec2 associate-route-table --route-table-id RTB-ID --subnet-id SUBNET-ID

# Elastic IPs
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id INSTANCE-ID --allocation-id ALLOCATION-ID
aws ec2 disassociate-address --association-id ASSOCIATION-ID
aws ec2 release-address --allocation-id ALLOCATION-ID
aws ec2 describe-addresses --output table
```

---

## S3 Commands

```bash
aws s3 ls                                          # List all buckets
aws s3 mb s3://bucket-name                         # Create bucket
aws s3 rb s3://bucket-name                         # Delete an empty bucket
aws s3 cp filename.txt s3://bucket-name            # Upload file
aws s3 cp s3://bucket-name/file.txt ./             # Download file
aws s3 rm s3://bucket-name/filename.txt            # Delete file
aws s3 ls s3://bucket-name                         # List bucket contents
aws s3 sync ./folder s3://bucket-name              # Sync a folder to S3

# Static website hosting
aws s3 website s3://bucket-name --index-document index.html

# Disable block public access (needed before a public bucket policy)
aws s3api put-public-access-block --bucket BUCKET \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Add a public-read bucket policy
aws s3api put-bucket-policy --bucket BUCKET --policy \
  '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::BUCKET/*"}]}'
```

---

## Lambda and API Gateway

Lambda and API Gateway are primarily configured through the console in this lab (trigger wiring, CORS settings), but key CLI equivalents:

```bash
aws lambda list-functions
aws lambda get-function --function-name my-first-function
aws lambda invoke --function-name my-first-function output.json
```

---

## Load Balancer Commands

```bash
# Create an Application Load Balancer
aws elbv2 create-load-balancer --name my-lab-alb \
  --subnets SUBNET-ID-1 SUBNET-ID-2 SUBNET-ID-3 \
  --security-groups SG-ID --scheme internet-facing --type application

aws elbv2 describe-load-balancers --names my-lab-alb

# Target groups
aws elbv2 create-target-group --name my-lab-targets --protocol HTTP --port 80 \
  --vpc-id VPC-ID --health-check-path / --health-check-interval-seconds 30

aws elbv2 register-targets --target-group-arn TARGET-GROUP-ARN \
  --targets Id=INSTANCE-ID-1 Id=INSTANCE-ID-2

aws elbv2 describe-target-health --target-group-arn TARGET-GROUP-ARN

# Listener
aws elbv2 create-listener --load-balancer-arn LB-ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=TARGET-GROUP-ARN
```

---

## CloudWatch and SNS Commands

```bash
# List all alarms
aws cloudwatch describe-alarms --region us-east-2

# Billing alarm — MUST be created in us-east-1
aws cloudwatch put-metric-alarm --alarm-name "AWS-Billing-Alert" \
  --metric-name EstimatedCharges --namespace AWS/Billing \
  --statistic Maximum --period 86400 --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD --evaluation-periods 1 \
  --alarm-actions SNS-ARN --treat-missing-data notBreaching --region us-east-1

# CPU alarm — same region as the EC2 instance
aws cloudwatch put-metric-alarm --alarm-name "EC2-CPU-High" \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=INSTANCE-ID --evaluation-periods 2 \
  --alarm-actions SNS-ARN --treat-missing-data notBreaching --region us-east-2

# Delete an alarm
aws cloudwatch delete-alarms --alarm-names "ALARM-NAME" --region us-east-2

# SNS
aws sns create-topic --name topic-name --region REGION
aws sns subscribe --topic-arn TOPIC-ARN --protocol email --notification-endpoint EMAIL --region REGION
aws sns list-subscriptions --region REGION
aws sns publish --topic-arn TOPIC-ARN --message "Test message" --region REGION
```

---

## CloudTrail Commands

```bash
aws cloudtrail describe-trails --region us-east-2
aws cloudtrail create-trail --name TRAIL-NAME --s3-bucket-name BUCKET --is-multi-region-trail
aws cloudtrail start-logging --name TRAIL-NAME
aws cloudtrail stop-logging --name TRAIL-NAME
aws cloudtrail get-trail-status --name TRAIL-NAME
aws cloudtrail lookup-events --max-results 10 --region us-east-2
```

---

## IAM Commands

```bash
aws iam list-users
aws iam list-roles
aws iam create-user --user-name USERNAME
aws iam attach-user-policy --user-name USERNAME --policy-arn arn:aws:iam::aws:policy/POLICY-NAME
aws iam list-attached-user-policies --user-name USERNAME
aws iam create-access-key --user-name USERNAME
aws iam delete-access-key --user-name USERNAME --access-key-id KEY-ID
aws sts get-caller-identity
```

---

## General / Cross-Cutting Commands

```bash
aws configure                          # Configure default credentials/region
aws configure --profile NAME           # Configure a named profile (e.g. for least-privilege testing)
aws configure get region
aws sts get-caller-identity
aws ec2 help
aws s3 help
aws --version
```

---

## Linux Commands Used on EC2

```bash
cat /etc/os-release          # OS version
whoami                        # Current user
free -h                       # Memory usage
df -h                         # Disk space
ip addr                       # Network interfaces
top                           # Running processes (q to exit)
sudo dnf install PKG -y       # Install a package
sudo systemctl start SVC      # Start a service
sudo systemctl enable SVC     # Enable service on boot
sudo systemctl status SVC     # Check service status
sudo bash -c 'echo "text" > /path/file'   # Write a file as root
```

---

## Windows PowerShell Commands

```powershell
# Navigate and connect
cd "C:\AWS LAB"
ssh -i "my-lab-key.pem" ec2-user@PUBLIC-IP

# Fix .pem file permissions (required before first SSH use)
takeown /F "C:\AWS LAB\my-lab-key.pem"
icacls "C:\AWS LAB\my-lab-key.pem" /inheritance:r /remove "BUILTIN\Users" /remove "Everyone"

# Find a misplaced .pem file
Get-ChildItem -Path C:\ -Recurse -Filter "*.pem" 2>$null

# AWS CLI on Windows
aws --version
aws configure
aws sts get-caller-identity
```

---

## Common ARN Formats

```
EC2 Instance:    arn:aws:ec2:REGION:ACCOUNT:instance/INSTANCE-ID
S3 Bucket:       arn:aws:s3:::BUCKET-NAME
IAM User:        arn:aws:iam::ACCOUNT:user/USERNAME
Lambda:          arn:aws:lambda:REGION:ACCOUNT:function:FUNCTION-NAME
Load Balancer:   arn:aws:elasticloadbalancing:REGION:ACCOUNT:loadbalancer/app/NAME/ID
Target Group:    arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/NAME/ID
CloudTrail:      arn:aws:cloudtrail:REGION:ACCOUNT:trail/TRAIL-NAME
SNS Topic:       arn:aws:sns:REGION:ACCOUNT:TOPIC-NAME
```

---

## EC2 Instance State Codes

| Code | State |
|---|---|
| 0 | Pending |
| 16 | Running |
| 32 | Shutting down |
| 48 | Terminated |
| 64 | Stopping |
| 80 | Stopped |

---

## Free Tier Limits

| Service | Free Tier |
|---|---|
| EC2 t3.micro | 750 hours/month for 12 months |
| S3 | 5GB storage, 20K GET, 2K PUT for 12 months |
| Lambda | 1M requests/month forever |
| CloudWatch | 10 metrics, 10 alarms free for 12 months |
| SNS | 1M notifications free for 12 months |
| CloudTrail | First trail free; additional trails and data events cost extra |
| CloudShell | Always free |
| Elastic IP | Free when attached to a running instance |

---

*Last updated: July 10, 2026*
