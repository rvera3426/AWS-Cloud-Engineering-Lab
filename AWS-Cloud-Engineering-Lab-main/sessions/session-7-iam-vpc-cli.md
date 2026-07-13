Session 7 — IAM Least Privilege, VPC Subnets, and Local CLI

July 10, 2026

What Was Built


Least-privilege IAM user, verified with positive and negative access tests
A new private VPC subnet with a dedicated route table
AWS CLI v2 installed and configured natively on Windows



Part 1 — IAM Least Privilege (Built and Tested)

Created the User and Attached a Read-Only Policy

bashaws iam create-user --user-name s3-readonly-user
aws iam attach-user-policy --user-name s3-readonly-user --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam list-attached-user-policies --user-name s3-readonly-user

Tested the Policy with a Separate CLI Profile

bashaws configure --profile s3readonly

# Test 1 — list bucket contents: SUCCEEDED
aws s3 ls s3://ramon-aws-lab-2026 --profile s3readonly

# Test 2 — upload a file: DENIED
aws s3 cp test.txt s3://ramon-aws-lab-2026/test.txt --profile s3readonly
# Result: AccessDenied — not authorized to perform s3:PutObject

# Test 3 — describe EC2 instances: DENIED
aws ec2 describe-instances --profile s3readonly
# Result: UnauthorizedOperation — not authorized to perform ec2:DescribeInstances

TestExpectedActual ResultList S3 bucket contentsAllowedSucceededUpload file to S3DeniedAccessDeniedDescribe EC2 instancesDeniedUnauthorizedOperation

Cleaned Up

bashaws iam delete-access-key --user-name s3-readonly-user --access-key-id [KEY_ID]

Access keys should be deleted immediately once testing is complete.


Part 2 — Public vs Private VPC Subnets (Built and Verified)

Examined the Existing (Public) Route Table

bashaws ec2 describe-route-tables --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]'

Result showed 0.0.0.0/0 → igw-001b2b42d34a20192 — confirming all three original subnets are public because their route table sends internet-bound traffic to an Internet Gateway.

Built a New Private Subnet

bashaws ec2 create-subnet --vpc-id vpc-0b4590f07ebb14c5b --cidr-block 172.31.48.0/20 --availability-zone us-east-2a

Created a Dedicated Route Table with No Internet Route

bashaws ec2 create-route-table --vpc-id vpc-0b4590f07ebb14c5b

The new route table only contained the local VPC route — no 0.0.0.0/0 entry.

Associated the Route Table with the New Subnet

bashaws ec2 associate-route-table --route-table-id [ROUTE_TABLE_ID] --subnet-id [SUBNET_ID]
aws ec2 describe-route-tables --route-table-ids [ROUTE_TABLE_ID]

Confirmed: no internet gateway route exists in this table.

Public Subnet (Original)Private Subnet (Built Today)Local route (172.31.0.0/16)YesYesInternet Gateway route (0.0.0.0/0)YesNoReachable from internetYesNo


Part 3 — AWS CLI Installed Locally on Windows

powershell# Downloaded and installed https://awscli.amazonaws.com/AWSCLIV2.msi

aws --version
# Result: aws-cli/2.35.20 Python/3.14.6 Windows/11 exe/AMD64

aws configure
# Configured with a fresh access key generated for ramon-admin

aws sts get-caller-identity
# Confirmed identity: ramon-admin

aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output table
# Successfully controlled the AWS account directly from Windows PowerShell

Key Takeaway

A subnet's public/private status is determined entirely by its route table, not by any subnet-level flag. Least privilege isn't just a policy attached to a user — it's something you can and should actively verify with both positive tests (what should work) and negative tests (what should fail). Local CLI access removes any dependency on a browser for day-to-day AWS management.
