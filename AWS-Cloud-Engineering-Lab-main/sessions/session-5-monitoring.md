Session 5 — Styled Websites, SNS, and CloudWatch Alarms

June 28, 2026

What Was Built


Polished, styled websites for S3 landing page and both EC2 servers
Live Lambda API call rendered directly on the S3 page
SNS notification topics for billing and EC2 alerts
CloudWatch billing alarm and CPU alarms
Detailed EC2 monitoring (1-minute intervals)


Steps

1. Built Three Styled Web Pages


S3 landing page — dark terminal theme, calls the Lambda API live via XMLHttpRequest and renders the JSON response on the page (avoids browser CSP eval restrictions that fetch + JSON.parse triggered)
EC2 Server 1 — cyan accent, shows instance ID, private IP, AZ
EC2 Server 2 — purple accent, same layout, deployed via a temporary Elastic IP since it had no public IP


2. Configured CORS

Enabled CORS on the API Gateway route and added Access-Control-Allow-Origin headers directly in the Lambda response so the browser-based S3 page could call the API cross-origin.

3. Created SNS Notification Topics

bash# Billing alerts must live in us-east-1 (billing metrics are only available there)
aws sns create-topic --name billing-alert --region us-east-1
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:[ACCOUNT_ID]:billing-alert \
  --protocol email --notification-endpoint [your-email] --region us-east-1

# EC2 alerts live in us-east-2, matching the instances' region
aws sns create-topic --name ec2-alerts --region us-east-2
aws sns subscribe --topic-arn arn:aws:sns:us-east-2:[ACCOUNT_ID]:ec2-alerts \
  --protocol email --notification-endpoint [your-email] --region us-east-2

4. Enabled Detailed EC2 Monitoring

bashaws ec2 monitor-instances --instance-ids i-0351bbe8831d04f1e i-0ab56766b7468d9bd

Metrics now collected every 1 minute instead of the default 5.

5. Created a CloudWatch Billing Alarm

bashaws cloudwatch put-metric-alarm \
  --alarm-name "AWS-Billing-Alert" \
  --metric-name EstimatedCharges --namespace AWS/Billing \
  --statistic Maximum --period 86400 --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:[ACCOUNT_ID]:billing-alert \
  --treat-missing-data notBreaching --region us-east-1

6. Created CloudWatch CPU Alarms (Both Instances)

bashaws cloudwatch put-metric-alarm \
  --alarm-name "EC2-Server1-CPU-High" \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0351bbe8831d04f1e \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-2:[ACCOUNT_ID]:ec2-alerts \
  --treat-missing-data notBreaching --region us-east-2

(Repeated for the second instance.) Both showed baseline CPU around 0.33% — well under the 80% threshold.

Key Takeaway

CloudWatch monitors metrics and triggers alarms; SNS delivers the actual notification. Billing metrics are a special case — they only exist in us-east-1 regardless of where your resources are deployed, and the SNS topic used by an alarm must be in the same region as the alarm itself.
