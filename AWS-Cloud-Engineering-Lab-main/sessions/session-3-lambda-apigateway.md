Session 3 — Lambda and API Gateway

June 12, 2026

What Was Built


Python 3.12 Lambda function
API Gateway HTTP API connected to Lambda
Live public API endpoint tested from a browser


Steps

1. Created the Lambda Function

Lambda → Create function → Author from scratch → Python 3.12 → basic execution role.

2. Function Code

pythonimport json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': 'Hello from Lambda!',
            'name': 'Ramon Vera',
            'lab': 'AWS Cloud Lab',
            'status': 'running'
        })
    }

Deployed and tested — green "Execution result: succeeded."

3. Connected API Gateway

Lambda page → Add trigger → API Gateway → HTTP API → Security: Open.

Result: https://ycc7afva43.execute-api.us-east-2.amazonaws.com/default/my-first-function

Tested directly in a browser — returned live JSON response.

Execution Metrics Observed

MetricValueDuration2.10 msBilled Duration103 msMemory Used36 MB of 128 MB allocatedInit Duration (cold start)100.84 ms

Key Takeaway

Lambda is serverless compute — code runs only when triggered, with no server to manage or pay for while idle. The cold start (first invocation) takes ~100ms to spin up a container; subsequent "warm" invocations run in single-digit milliseconds. API Gateway is the front door that turns a Lambda function into a public HTTP API.
