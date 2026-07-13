Session 1 — Account Setup and First EC2 Instance

June 9, 2026

What Was Built


AWS Free Tier account
MFA enabled on root account
IAM user with least-privilege practice (ramon-admin)
First EC2 instance (Amazon Linux 2023, t3.micro)
Elastic IP for a persistent public address
SSH access from Windows PowerShell and browser-based CloudShell
First S3 bucket


Steps

1. Created AWS Free Tier Account

Signed up at aws.amazon.com/free and selected the Basic Support (Free) plan.

2. Secured the Root Account with MFA

IAM → Security Credentials → Assign MFA device, using an authenticator app.
Why: The root account has unlimited access — it must be protected and never used for daily tasks.

3. Created an IAM User

bash# Done via console: IAM → Users → Create User → ramon-admin
# Attached AdministratorAccess policy
# Downloaded credentials .csv

Why: Never use root for day-to-day work — principle of least privilege.

4. Launched First EC2 Instance


EC2 → Launch Instance → Amazon Linux 2023 → t3.micro (free tier eligible)
Created key pair my-lab-key, saved to C:\AWS LAB\my-lab-key.pem


5. Assigned an Elastic IP

EC2 → Network & Security → Elastic IPs → Allocate → Associate.
Result: 3.150.25.101
Why: Without an Elastic IP, the public IP changes every time the instance stops/starts.

6. Fixed .pem File Permissions on Windows

powershelltakeown /F "C:\AWS LAB\my-lab-key.pem"
icacls "C:\AWS LAB\my-lab-key.pem" /inheritance:r /remove "BUILTIN\Users" /remove "Everyone"

SSH refuses to use a private key that is readable by other users — this enforces that.

7. Connected via SSH and CloudShell

powershellcd "C:\AWS LAB"
ssh -i "my-lab-key.pem" ec2-user@3.150.25.101

8. First AWS CLI Commands

bashaws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table
aws s3 mb s3://ramon-aws-lab-2026
aws ec2 stop-instances --instance-ids i-0351bbe8831d04f1e

Key Takeaway

EC2 is a rented virtual machine — same concept as a physical server, except hosted in AWS's data center and billed by the hour. Elastic IP solves the problem of public IPs changing on every restart.
