# AWS S3 + Lambda + SHA-256 + DynamoDB
## Step-by-Step Duplicate File Detection Project

## 1. Project Overview

This project detects duplicate files uploaded to Amazon S3 by calculating their SHA-256 fingerprints.

When a user uploads a file:

1. Amazon S3 stores the uploaded file.
2. S3 generates an ObjectCreated event.
3. AWS Lambda receives the event.
4. Lambda downloads and reads the file contents.
5. Python calculates the SHA-256 hash.
6. Lambda checks DynamoDB for an existing hash.
7. If the hash exists, duplicate processing is skipped.
8. If the hash does not exist, the file is registered and processed.

### Architecture

```text
                User Uploads File
                       |
                       v
                 Amazon S3
                       |
                       | ObjectCreated Event
                       v
                  AWS Lambda
                       |
                       | Download File
                       v
                Read File Bytes
                       |
                       v
                SHA-256 Hashing
                       |
                       v
                  DynamoDB
                       |
                 Check Hash
                       |
              +--------+--------+
              |                 |
              v                 v
         Hash Exists       Hash Not Found
              |                 |
              v                 v
        Duplicate File       New File
              |                 |
              v                 v
       Skip Processing     Save Hash
                                |
                                v
                         Process File
                                |
                                v
                         CloudWatch Logs
```

## 2. AWS Services Used

| Service | Purpose |
|---|---|
| Amazon S3 | Stores uploaded files |
| AWS Lambda | Reads files and calculates hashes |
| Python hashlib | Calculates SHA-256 |
| Amazon DynamoDB | Stores previously processed hashes |
| AWS IAM | Manages permissions |
| Amazon CloudWatch | Stores Lambda logs |

## 3. Prerequisites

Install:

- AWS CLI
- Python 3
- VS Code
- AWS account

Verify Python:

```bash
python3 --version
```

Verify AWS CLI:

```bash
aws --version
```

Check AWS identity:

```bash
aws sts get-caller-identity
```

Configure AWS CLI if necessary:

```bash
aws configure
```

Example region:

```text
ap-south-1
```

## 4. Create Project Directory

```bash
mkdir s3-duplicate-detection

cd s3-duplicate-detection
```

Create a virtual environment:

```bash
python3 -m venv .venv
```

Activate:

```bash
source .venv/bin/activate
```

Create files:

```bash
touch file1.txt
touch file2.txt
touch file3.txt
touch hash_demo.py
```

Project structure:

```text
s3-duplicate-detection/
│
├── .venv/
├── file1.txt
├── file2.txt
├── file3.txt
├── hash_demo.py
├── lambda_function.py
└── README.md
```

## 5. Understand How Files Are Stored

Computers store file contents as bytes.

Example:

```text
Hello AWS
```

Its UTF-8 hexadecimal representation is:

```text
48 65 6c 6c 6f 20 41 57 53
```

A SHA-256 algorithm takes the bytes and generates a fixed-length fingerprint.

Important SHA-256 properties:

| Property | Value |
|---|---|
| Output size | 256 bits |
| Output bytes | 32 bytes |
| Hexadecimal characters | 64 |
| Same input | Same hash |
| Different input | Normally different hash |

## 6. Step 1: Create Sample Files

Add the same content to two files:

```bash
echo -n "Hello AWS" > file1.txt

echo -n "Hello AWS" > file2.txt
```

Add different content to the third file:

```bash
echo -n "Hello Cloud" > file3.txt
```

Check the content:

```bash
cat file1.txt

cat file2.txt

cat file3.txt
```

Expected:

```text
file1.txt = Hello AWS
file2.txt = Hello AWS
file3.txt = Hello Cloud
```

## 7. Step 2: Calculate SHA-256 Using Terminal

Run:

```bash
shasum -a 256 file1.txt

shasum -a 256 file2.txt

shasum -a 256 file3.txt
```

Expected:

- `file1.txt` and `file2.txt` have identical hashes.
- `file3.txt` has a different hash.

This is your first duplicate-detection test.

## 8. Step 3: Calculate SHA-256 Using Python

Open `hash_demo.py`.

Add:

```python
import hashlib

def calculate_hash(file_name):

    with open(file_name, "rb") as file:

        content = file.read()

    file_hash = hashlib.sha256(
        content
    ).hexdigest()

    return file_hash


files = [
    "file1.txt",
    "file2.txt",
    "file3.txt"
]

for file_name in files:

    result = calculate_hash(file_name)

    print("File:", file_name)

    print("SHA-256:", result)

    print("----------------------")
```

Run:

```bash
python3 hash_demo.py
```

Expected result:

```text
File: file1.txt
SHA-256: <64-character hash>

File: file2.txt
SHA-256: <same 64-character hash>

File: file3.txt
SHA-256: <different 64-character hash>
```

## 9. Step 4: Understand Duplicate Detection Logic

The logic is:

```python
if file1_hash == file2_hash:

    print("Duplicate file detected")

else:

    print("New file detected")
```

A more complete comparison:

```python
import hashlib

def calculate_hash(file_name):

    with open(file_name, "rb") as file:

        return hashlib.sha256(
            file.read()
        ).hexdigest()


hash1 = calculate_hash("file1.txt")

hash2 = calculate_hash("file2.txt")

hash3 = calculate_hash("file3.txt")


if hash1 == hash2:

    print("file1 and file2 are duplicates")


if hash1 != hash3:

    print("file3 contains different content")
```

## 10. Step 5: Create the DynamoDB Table

Table name:

```text
FileUploadRegistry
```

Partition key:

```text
file_hash
```

Partition key type:

```text
String
```

Create using AWS CLI:

```bash
aws dynamodb create-table \
  --table-name FileUploadRegistry \
  --attribute-definitions \
      AttributeName=file_hash,AttributeType=S \
  --key-schema \
      AttributeName=file_hash,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

Wait until the table becomes active:

```bash
aws dynamodb wait table-exists \
  --table-name FileUploadRegistry \
  --region ap-south-1
```

Verify:

```bash
aws dynamodb describe-table \
  --table-name FileUploadRegistry \
  --region ap-south-1
```

Example DynamoDB record:

```json
{
  "file_hash": "abc123...",
  "file_name": "invoice.pdf",
  "bucket_name": "my-upload-bucket",
  "status": "PROCESSED"
}
```

## 11. Step 6: Create the Lambda Function

Create:

```bash
touch lambda_function.py
```

Install no extra package for SHA-256: Python's `hashlib` is built in. The AWS Lambda Python runtime includes boto3.

Add:

```python
import boto3
import hashlib
import urllib.parse
import os

from botocore.exceptions import ClientError

s3 = boto3.client("s3")

dynamodb = boto3.resource("dynamodb")

table = dynamodb.Table(
    os.environ["DYNAMODB_TABLE"]
)


def lambda_handler(event, context):

    for record in event["Records"]:

        bucket = record["s3"]["bucket"]["name"]

        file_name = urllib.parse.unquote_plus(
            record["s3"]["object"]["key"]
        )

        print("Uploaded file:", file_name)

        # Read the actual S3 object content
        response = s3.get_object(
            Bucket=bucket,
            Key=file_name
        )

        file_hash = hashlib.sha256()

        # Calculate hash incrementally
        for chunk in response["Body"].iter_chunks(
            chunk_size=1024 * 1024
        ):

            if chunk:
                file_hash.update(chunk)

        sha256_hash = file_hash.hexdigest()

        print("SHA-256:", sha256_hash)

        try:

            # Register only if the hash does not exist
            table.put_item(
                Item={
                    "file_hash": sha256_hash,
                    "file_name": file_name,
                    "bucket_name": bucket,
                    "status": "PROCESSING"
                },
                ConditionExpression=(
                    "attribute_not_exists(file_hash)"
                )
            )

            print("New file detected")

            # Add actual file-processing logic here.

            table.update_item(
                Key={
                    "file_hash": sha256_hash
                },
                UpdateExpression="SET #s = :status",
                ExpressionAttributeNames={
                    "#s": "status"
                },
                ExpressionAttributeValues={
                    ":status": "PROCESSED"
                }
            )

            print("File processed successfully")

        except ClientError as error:

            if error.response["Error"]["Code"] == (
                "ConditionalCheckFailedException"
            ):

                print("Duplicate file detected")

                print("Processing skipped")

            else:

                raise

    return {
        "statusCode": 200,
        "body": "File duplicate check completed"
    }
```

### Important explanation

`get_object()` reads the file from S3.

`iter_chunks()` reads the file incrementally.

`hashlib.sha256()` calculates the fingerprint.

`put_item()` registers a new fingerprint.

`ConditionExpression` prevents a second invocation from inserting the same fingerprint.

## 12. Step 7: Configure Lambda Environment Variable

Create an environment variable:

```text
DYNAMODB_TABLE = FileUploadRegistry
```

This tells Lambda which DynamoDB table to use.

Example AWS CLI command:

```bash
aws lambda update-function-configuration \
  --function-name s3-upload-processor \
  --environment "Variables={DYNAMODB_TABLE=FileUploadRegistry}" \
  --region ap-south-1
```

## 13. Step 8: Configure IAM Permissions

The Lambda execution role needs permission to read S3 objects and access DynamoDB.

### S3 policy

Replace the bucket name with your actual bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

### DynamoDB policy

Replace the account ID and region if necessary:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:YOUR-ACCOUNT-ID:table/FileUploadRegistry"
    }
  ]
}
```

The existing Lambda role should also retain its CloudWatch logging policy.

## 14. Step 9: Configure S3 Event Notification

Configure S3 to invoke Lambda when a new object is created.

The Lambda function must have a resource-based permission allowing the intended S3 bucket to invoke it.

Example:

```bash
aws lambda add-permission \
  --function-name s3-upload-processor \
  --statement-id s3-duplicate-detection \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::YOUR-BUCKET-NAME \
  --source-account YOUR-ACCOUNT-ID \
  --region ap-south-1
```

Then configure the bucket notification for:

```text
s3:ObjectCreated:*
```

**Important:** Preserve existing bucket notifications when updating the notification configuration. S3 event notifications are sent after an object is created, not before the upload.

Also, avoid triggering the same Lambda from its own output bucket or use separate input/output prefixes.

## 15. Step 10: Test Duplicate Detection

### Test 1: Upload first file

```bash
aws s3 cp file1.txt s3://YOUR-BUCKET-NAME/
```

Expected CloudWatch logs:

```text
Uploaded file: file1.txt
SHA-256: <hash>
New file detected
File processed successfully
```

DynamoDB stores the fingerprint.

### Test 2: Upload identical content with another filename

```bash
aws s3 cp file2.txt s3://YOUR-BUCKET-NAME/
```

Expected:

```text
Uploaded file: file2.txt
SHA-256: <same hash>
Duplicate file detected
Processing skipped
```

### Test 3: Upload different content

```bash
aws s3 cp file3.txt s3://YOUR-BUCKET-NAME/
```

Expected:

```text
Uploaded file: file3.txt
SHA-256: <different hash>
New file detected
File processed successfully
```

## 16. Step 11: Verify DynamoDB Records

Run:

```bash
aws dynamodb scan \
  --table-name FileUploadRegistry \
  --region ap-south-1
```

Expected:

Only one record for each unique file content hash.

For example:

```text
file1.txt → HASH A
file2.txt → HASH A
file3.txt → HASH B
```

DynamoDB stores:

```text
HASH A → PROCESSED
HASH B → PROCESSED
```

The duplicate file does not create another hash record.

## 17. Important Real-World Scenarios

| Scenario | Expected behavior |
|---|---|
| Same file, same filename | S3 may overwrite the object |
| Same file, different filename | SHA-256 detects duplicate content |
| Different file content | New SHA-256 hash |
| Same filename, different content | Different SHA-256 hash |
| Simultaneous duplicate uploads | DynamoDB conditional write allows only one initial claim |
| Lambda execution failure | Processing status and recovery need to be managed |

### Important production improvement

This is a basic demonstration of duplicate detection, not a complete production workflow.

For production:

- Use a processing lease or state machine for recovery.
- Handle S3 event retries and duplicate events.
- Store object version IDs when versioning is enabled.
- Do not mark a file PROCESSED until the actual business processing succeeds.
- Keep a separate file-upload record if you need to track every upload, including duplicates.
- Use a durable work queue such as SQS for reliable asynchronous processing.

## 18. Important Interview Questions

**Q1. Does S3 automatically detect duplicate files?**

No. S3 identifies objects by bucket and key. Additional application logic is needed to detect identical content.

**Q2. What is SHA-256?**

A cryptographic hash function that converts file bytes into a 256-bit fingerprint.

**Q3. Why use SHA-256 instead of filename comparison?**

Two files can have different names but identical contents. A content hash can identify this.

**Q4. Why use DynamoDB?**

It provides persistent storage for fingerprints and supports conditional writes to prevent duplicate registrations.

**Q5. What is the difference between ETag and SHA-256?**

ETag is an S3 object identifier/check value with behavior that varies by upload and encryption configuration. It is not universally a content MD5 checksum or SHA-256 hash.

**Q6. Can Lambda prevent the S3 upload?**

An S3 event-triggered Lambda runs after the object is created. To reject uploads before storage, use an API or upload service that checks the content before allowing the upload.

**Q7. How do you calculate a hash for a large file?**

Read the object in chunks and update the SHA-256 hash incrementally instead of loading the entire object into memory.

## 19. Final Project Architecture

```text
                 React Application
                        |
                        v
                  API Gateway
                        |
                        v
                  Upload Lambda
                        |
                        v
                    S3 Bucket
                        |
                        v
                 S3 Event Trigger
                        |
                        v
                 Processing Lambda
                        |
                        v
                 Read File Bytes
                        |
                        v
                  SHA-256 Hash
                        |
                        v
                   DynamoDB
                        |
              +---------+---------+
              |                   |
              v                   v
         Hash Exists         Hash Not Found
              |                   |
              v                   v
       Skip Duplicate       Process New File
                                  |
                                  v
                           CloudWatch Logs
```

## Conclusion

You have learned how to build a content-based duplicate-detection workflow using:

```text
Amazon S3
AWS Lambda
Python hashlib
SHA-256
Amazon DynamoDB
AWS IAM
Amazon CloudWatch
```

This is useful for document management systems, image upload platforms, invoice processing, backup systems and enterprise data ingestion pipelines.

**Key concept to remember:**

S3 stores the file. Lambda reads the actual bytes. SHA-256 generates the fingerprint. DynamoDB remembers the fingerprint and helps identify duplicate content.
