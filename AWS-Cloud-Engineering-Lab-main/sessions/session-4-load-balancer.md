Session 4 — Application Load Balancer

June 22, 2026

What Was Built


Second EC2 instance, auto-configured via User Data (no manual SSH)
Application Load Balancer spanning 3 Availability Zones
Target group with health checks
Verified round-robin traffic distribution


Steps

1. Found the Correct AMI for the Region

bashaws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*" "Name=architecture,Values=x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text

2. Launched Second EC2 Instance with User Data

bashaws ec2 run-instances \
  --image-id ami-0741dc526e1106ae5 \
  --instance-type t3.micro \
  --key-name my-lab-key \
  --security-group-ids sg-0e7ae38cc5f20b59c \
  --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Server 2</h1>" > /var/www/html/index.html' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MySecondServer}]'

Apache was installed and running automatically at first boot — no manual SSH required.

3. Created a Dedicated Security Group for the Load Balancer

bashaws ec2 create-security-group --group-name my-load-balancer-sg --description "LB SG" --vpc-id vpc-0b4590f07ebb14c5b
aws ec2 authorize-security-group-ingress --group-id sg-0d3241a1bff4666f9 --protocol tcp --port 80 --cidr 0.0.0.0/0

4. Created the Application Load Balancer

bashaws elbv2 create-load-balancer \
  --name my-lab-alb \
  --subnets subnet-03340437b0d4d94aa subnet-03d61ea4c4ad0a028 subnet-04dd784ef6b5d6d12 \
  --security-groups sg-0d3241a1bff4666f9 \
  --scheme internet-facing \
  --type application

Spans three Availability Zones: us-east-2a, us-east-2b, us-east-2c.

5. Created a Target Group and Registered Both Instances

bashaws elbv2 create-target-group --name my-lab-targets --protocol HTTP --port 80 \
  --vpc-id vpc-0b4590f07ebb14c5b --health-check-path / --health-check-interval-seconds 30

aws elbv2 register-targets --target-group-arn [TARGET_GROUP_ARN] \
  --targets Id=i-0351bbe8831d04f1e Id=i-0ab56766b7468d9bd

6. Created the Listener

bashaws elbv2 create-listener --load-balancer-arn [LB_ARN] --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=[TARGET_GROUP_ARN]

7. Verified Health and Tested Load Balancing

bashaws elbv2 describe-target-health --target-group-arn [TARGET_GROUP_ARN]

Both instances showed State: healthy. Refreshing the load balancer URL in a browser alternated between Server 1 and Server 2, confirming round-robin distribution.

Key Takeaway

An Application Load Balancer distributes traffic across multiple targets and performs health checks every 30 seconds — if an instance fails a check, the ALB automatically stops routing traffic to it. Combined with multi-AZ deployment, this is the standard pattern for high availability.
