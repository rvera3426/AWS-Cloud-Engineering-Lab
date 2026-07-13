Session 2 — Web Server and S3 Static Website

June 12, 2026

What Was Built


Apache web server on EC2
Custom HTML page served over HTTP
S3 bucket configured for static website hosting


Steps

1. Installed and Started Apache

bashsudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd

2. Opened Port 80 in the Security Group

bashaws ec2 authorize-security-group-ingress --group-id sg-0e7ae38cc5f20b59c --protocol tcp --port 80 --cidr 0.0.0.0/0

Result: http://3.150.25.101 served a live webpage.

3. Hosted a Static Website on S3

bashaws s3 cp index.html s3://ramon-aws-lab-2026/index.html
aws s3 website s3://ramon-aws-lab-2026 --index-document index.html

aws s3api put-public-access-block --bucket ramon-aws-lab-2026 \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

aws s3api put-bucket-policy --bucket ramon-aws-lab-2026 --policy \
  '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::ramon-aws-lab-2026/*"}]}'

Result: http://ramon-aws-lab-2026.s3-website.us-east-2.amazonaws.com

Key Takeaway

Two different ways to serve a website on AWS: EC2 requires a running server and web server software (Apache); S3 static hosting requires neither — it serves files directly from storage. EC2 is better for dynamic apps, S3 is better for static content at near-zero cost.
