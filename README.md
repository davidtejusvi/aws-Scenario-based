# aws-Scenario-based

# AWS S3 → Lambda Event-Driven File Processing

## 1. Project Overview

This project demonstrates how to automatically trigger an AWS Lambda function when a file is uploaded to an Amazon S3 bucket.

When a user uploads a file to S3:

1. Amazon S3 detects the file upload event.
2. S3 sends an event notification to AWS Lambda.
3. Lambda executes the Python function.
4. Lambda reads the event details.
5. Lambda writes execution information to Amazon CloudWatch Logs.

### Architecture

```text
             User / Developer
                    |
                    | Upload file
                    v
             Amazon S3 Bucket
                    |
                    | ObjectCreated Event
                    v
              AWS Lambda
             Python 3.12
                    |
                    | Execution logs
                    v
             CloudWatch Logs
```

### AWS Services Used

| Service | Purpose |
|---|---|
| Amazon S3 | Stores uploaded files |
| AWS Lambda | Executes Python code |
| AWS IAM | Controls permissions |
| CloudWatch | Stores Lambda execution logs |
| AWS CLI | Creates and manages AWS resources |

## 2. Project Configuration

The following configuration was used during development.

| Configuration | Value |
|---|---|
| AWS Region | `ap-south-1` |
| AWS Account | `919006484182` |
| IAM User | `terraform-user` |
| Lambda Function | `s3-upload-processor` |
| Runtime | Python 3.12 |
| Handler | `lambda_function.lambda_handler` |
| IAM Execution Role | `lambda-s3-execution-role` |
| S3 Bucket | Needs verification |

**Important:** The bucket name `david-s3-upload-processer-1234455677` was attempted during testing but returned `NoSuchBucket`. Verify the actual bucket before configuring the trigger.

## 3. Prerequisites

Install and configure:

- AWS CLI
- Python 3
- Visual Studio Code
- An AWS account
- An IAM user with permissions to create the required resources

Verify AWS CLI:

```bash
aws --version
```

Verify Python:

```bash
python3 --version
```

Configure AWS credentials:

```bash
aws configure
```

Provide:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-south-1
Default output format: json
```

Verify the AWS identity:

```bash
aws sts get-caller-identity
```

## 4. Create the Project Directory

Create a project directory:

```bash
mkdir s3-lambda-project

cd s3-lambda-project
```

Create a Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Create the Lambda Python file:

```bash
touch lambda_function.py
```

Project structure:

```text
s3-lambda-project/
│
├── .venv/
│
├── lambda_function.py
│
├── function.zip
│
├── test_event.json
│
├── local_test.py
│
└── README.md
```

## 5. Write the Lambda Function

Open `lambda_function.py`.

Add this Python code:

```python
import json
import urllib.parse

def lambda_handler(event, context):

    print("S3 upload Event Received")

    for record in event['Records']:

        bucket = record['s3']['bucket']['name']

        file_name = urllib.parse.unquote_plus(
            record['s3']['object']['key']
        )

        file_size = record['s3']['object']['size']

        event_name = record['eventName']

        print("Bucket:", bucket)
        print("File Name:", file_name)
        print("File Size:", file_size)
        print("Event:", event_name)

    return {
        'statusCode': 200,
        'body': json.dumps('File processed successfully')
    }
```

### Explanation

| Python code | What it does |
|---|---|
| `lambda_handler()` | Entry point of the Lambda function |
| `event` | Receives the S3 event information |
| `context` | Provides Lambda runtime information |
| `event['Records']` | Contains event records |
| `bucket` | Gets the S3 bucket name |
| `file_name` | Gets the uploaded file name |
| `file_size` | Gets the uploaded file size |
| `eventName` | Identifies the S3 event type |
| `print()` | Writes information to CloudWatch Logs |

Example event:

```text
ObjectCreated:Put
```

This means an object was uploaded to S3 using a PUT operation.

## 6. Create the IAM Execution Role

Lambda needs an IAM role to write execution logs to CloudWatch.

Create the trust policy file:

```bash
nano trust-policy.json
```

Add:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role:

```bash
aws iam create-role \
  --role-name lambda-s3-execution-role \
  --assume-role-policy-document file://trust-policy.json
```

Attach the AWS-managed logging policy:

```bash
aws iam attach-role-policy \
  --role-name lambda-s3-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

The role allows Lambda to assume the role and write logs to CloudWatch.

Execution role ARN:

```text
arn:aws:iam::919006484182:role/lambda-s3-execution-role
```

## 7. Package the Lambda Function

Create the deployment ZIP:

```bash
zip function.zip lambda_function.py
```

Check the ZIP contents:

```bash
unzip -l function.zip
```

The ZIP must contain `lambda_function.py` at its root.

## 8. Create the Lambda Function

Create the Lambda function through AWS CLI:

```bash
aws lambda create-function \
  --function-name s3-upload-processor \
  --runtime python3.12 \
  --role arn:aws:iam::919006484182:role/lambda-s3-execution-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 128 \
  --region ap-south-1
```

Verify Lambda:

```bash
aws lambda get-function \
  --function-name s3-upload-processor \
  --region ap-south-1
```

Lambda ARN:

```text
arn:aws:lambda:ap-south-1:919006484182:function:s3-upload-processor
```

## 9. Test Lambda Manually

Before connecting S3, test Lambda independently.

Create the test event:

```bash
nano test_event.json
```

Add:

```json
{
  "Records": [
    {
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-test-bucket"
        },
        "object": {
          "key": "sample.txt",
          "size": 25
        }
      }
    }
  ]
}
```

Invoke Lambda:

```bash
aws lambda invoke \
  --function-name s3-upload-processor \
  --payload fileb://test_event.json \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region ap-south-1
```

Check the response:

```bash
cat response.json
```

Expected response:

```json
{
  "statusCode": 200,
  "body": "\"File processed successfully\""
}
```

### Check CloudWatch Logs

```bash
aws logs tail /aws/lambda/s3-upload-processor \
  --since 10m \
  --region ap-south-1
```

Expected output:

```text
S3 upload Event Received
Bucket: my-test-bucket
File Name: sample.txt
File Size: 25
Event: ObjectCreated:Put
```

**Result:** Manual Lambda invocation successfully executed the Python function and generated CloudWatch logs.

## 10. Create or Verify the S3 Bucket

List existing buckets:

```bash
aws s3 ls
```

Check the AWS account:

```bash
aws sts get-caller-identity
```

If the bucket does not exist, create a globally unique bucket name.

Example:

```bash
aws s3api create-bucket \
  --bucket YOUR-UNIQUE-BUCKET-NAME \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
```

Verify the bucket:

```bash
aws s3api head-bucket \
  --bucket YOUR-UNIQUE-BUCKET-NAME
```

Check the bucket's region:

```bash
aws s3api get-bucket-location \
  --bucket YOUR-UNIQUE-BUCKET-NAME
```

Use the exact bucket name returned by `aws s3 ls`.

## 11. Allow S3 to Invoke Lambda

S3 needs permission to invoke the Lambda function.

Run:

```bash
aws lambda add-permission \
  --function-name s3-upload-processor \
  --statement-id s3-upload-invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::YOUR-UNIQUE-BUCKET-NAME \
  --source-account 919006484182 \
  --region ap-south-1
```

This adds resource-based permission to Lambda.

**Important:** If the permission already exists, AWS may return `ResourceConflictException`. Check the existing policy instead of adding the same statement again.

## 12. Configure the S3 Event Notification

First, inspect existing notifications:

```bash
aws s3api get-bucket-notification-configuration \
  --bucket YOUR-UNIQUE-BUCKET-NAME \
  --region ap-south-1
```

Configure S3 to trigger Lambda when an object is created:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket YOUR-UNIQUE-BUCKET-NAME \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [
      {
        "Id": "s3-upload-trigger",
        "LambdaFunctionArn": "arn:aws:lambda:ap-south-1:919006484182:function:s3-upload-processor",
        "Events": ["s3:ObjectCreated:*"]
      }
    ]
  }' \
  --region ap-south-1
```

**Important:** This command replaces the bucket's entire notification configuration. Preserve any existing Lambda, SNS, SQS or EventBridge notifications when updating it.

## 13. Real-Time S3 Upload Test

Open Terminal 1 and follow Lambda logs:

```bash
aws logs tail /aws/lambda/s3-upload-processor \
  --since 1m \
  --follow \
  --region ap-south-1
```

Open Terminal 2.

Create a file:

```bash
echo "Hello from real S3 upload" > realtime.txt
```

Upload it:

```bash
aws s3 cp realtime.txt \
  s3://YOUR-UNIQUE-BUCKET-NAME/
```

Expected CloudWatch output:

```text
S3 upload Event Received
Bucket: YOUR-UNIQUE-BUCKET-NAME
File Name: realtime.txt
File Size: 26
Event: ObjectCreated:Put
```

If this output appears after uploading the file, the automatic S3-to-Lambda integration is working.

## 14. Troubleshooting

### Error: NoSuchBucket

Example:

```text
An error occurred (NoSuchBucket)
The specified bucket does not exist.
```

Possible reasons:

- Incorrect bucket name.
- Bucket was never created.
- AWS CLI is using a different AWS account.
- Bucket was deleted.

Solutions:

```bash
aws sts get-caller-identity

aws s3 ls
```

Copy the exact bucket name and retry.

### Error: AccessDenied

Check:

- IAM user permissions.
- Lambda execution role permissions.
- Lambda resource-based permission.
- S3 bucket policy, if applicable.

### Lambda does not trigger

Check:

1. S3 notification configuration.
2. Lambda resource-based permission.
3. Correct bucket ARN.
4. Correct Lambda ARN.
5. Matching AWS Region.
6. CloudWatch Logs.

## 15. Important Interview Concepts

| Interview question | Explanation |
|---|---|
| What is S3 event notification? | It notifies another AWS service when an S3 event occurs. |
| How does S3 trigger Lambda? | S3 invokes Lambda through an event notification. |
| What is an IAM execution role? | It provides permissions to the Lambda function. |
| What is a Lambda resource policy? | It controls which services or accounts can invoke Lambda. |
| What is CloudWatch? | It collects logs and monitoring information. |
| What is `ObjectCreated`? | An S3 event generated when an object is created. |
| What is a Lambda handler? | The function that AWS Lambda executes. |
| What is `event` in Lambda? | The input data passed to the function. |

## 16. Security Best Practices

- Keep S3 Block Public Access enabled unless public access is specifically required.
- Use IAM roles instead of hardcoding AWS access keys.
- Follow least-privilege IAM permissions.
- Enable encryption for S3 objects.
- Do not store credentials in Python source code.
- Restrict S3-to-Lambda invocation to the intended bucket.
- Monitor Lambda errors using CloudWatch.
- Make processing idempotent because S3 event notifications can be delivered more than once.

## 17. Current Project Completion Status

| Task | Status |
|---|---|
| AWS CLI setup | Completed |
| Python virtual environment | Completed |
| Lambda Python code | Completed |
| IAM trust policy | Completed |
| IAM execution role | Created |
| CloudWatch logging policy | Attached |
| Lambda ZIP package | Created |
| Lambda deployment | Completed |
| Manual Lambda test | Successful |
| CloudWatch log verification | Successful |
| S3 bucket verification | Pending |
| S3-to-Lambda permission | Must verify |
| S3 event notification | Must verify |
| Real-time S3 upload test | Pending |

## 18. Future Improvements

After the basic integration works, extend the project:

1. Read uploaded file contents using `boto3` and `s3.get_object()`.
2. Add `s3:GetObject` permission to the Lambda execution role.
3. Process uploaded CSV or JSON files.
4. Store processed results in another S3 bucket.
5. Add SNS notifications for successful or failed processing.
6. Add error handling and retry strategies.
7. Deploy the infrastructure using Terraform.
8. Automate deployment using GitHub Actions.

### Final Architecture

```text
                 Developer
                     |
                     | Upload file
                     v
              Amazon S3 Bucket
                     |
                     | S3 Event Notification
                     v
                AWS Lambda
                     |
              Python Processing
                     |
              +------+------+
              |             |
              v             v
          CloudWatch     Destination S3
             Logs        (Future Upgrade)
```

## Conclusion

This project demonstrates an event-driven serverless architecture using Amazon S3, AWS Lambda, IAM and CloudWatch.

The Lambda function has been deployed and successfully tested through manual invocation. The remaining step is to verify the correct S3 bucket, configure the event notification and confirm that a real file upload automatically invokes Lambda.
