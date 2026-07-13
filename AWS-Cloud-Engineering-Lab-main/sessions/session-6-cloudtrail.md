Session 6 — CloudTrail

July 9, 2026

What Was Built


Dedicated S3 bucket for CloudTrail logs
Multi-region CloudTrail trail
Verified active logging and queried real API events


Steps

1. Checked for an Existing Trail

bashaws cloudtrail describe-trails --region us-east-2
# Result: empty trailList — no trail existed yet

2. Created a Dedicated S3 Bucket for Logs

bashaws s3 mb s3://ramon-cloudtrail-logs-2026

3. Attached a Bucket Policy Allowing CloudTrail to Write

bashaws s3api put-bucket-policy --bucket ramon-cloudtrail-logs-2026 --policy '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::ramon-cloudtrail-logs-2026"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::ramon-cloudtrail-logs-2026/AWSLogs/[ACCOUNT_ID]/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}'

4. Created a Multi-Region Trail

bashaws cloudtrail create-trail --name my-lab-trail --s3-bucket-name ramon-cloudtrail-logs-2026 --is-multi-region-trail

A multi-region trail captures API activity across all AWS regions, not just the one it was created in.

5. Started Logging

bashaws cloudtrail start-logging --name my-lab-trail
aws cloudtrail get-trail-status --name my-lab-trail
# Result: IsLogging: true

6. Verified Events via Lookup

bashaws cloudtrail lookup-events --max-results 5 --region us-east-2

Real events captured included CreateTrail, GetCallerIdentity, and GetBucketAcl — each entry showing the exact IAM identity (ramon-admin), source IP address, timestamp, and full API request/response detail.

7. Confirmed Log Delivery to S3

bashaws s3 ls s3://ramon-cloudtrail-logs-2026/AWSLogs/ --recursive

A compressed .json.gz log file appeared within minutes of enabling logging.

Key Takeaway

CloudTrail is the audit trail for an entire AWS account. It answers "who did what, when, and from where" — every API call is logged, including CloudTrail logging its own creation event. This is the foundation for security investigations, and in a production environment these logs are typically shipped to a SIEM (e.g., Splunk) for detection and alerting — a natural bridge to the Splunk SOC Lab project.
