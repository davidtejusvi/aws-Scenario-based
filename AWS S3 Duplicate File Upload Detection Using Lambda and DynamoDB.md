# AWS S3 Duplicate File Upload Detection Using Lambda and DynamoDB

## 1. Project Overview

This project detects whether a file has already been uploaded to an S3 bucket.

If the same file is uploaded again, the system identifies it as a duplicate and avoids processing it again.

We use:

- Amazon S3 – Stores uploaded files.
- AWS Lambda – Detects and processes upload events.
- Amazon DynamoDB – Stores file fingerprints and processing status.
- AWS IAM – Provides secure permissions.
- Amazon CloudWatch – Stores execution logs.

## 2. How Does S3 Detect Duplicate Files?

Amazon S3 does not automatically prevent duplicate uploads.

For example:

```text
First upload:
report.pdf → Uploaded successfully

Second upload:
report.pdf → Existing object overwritten
```

S3 identifies an object primarily by its bucket and object key.

To detect duplicate content, we calculate a file fingerprint.

A common method is SHA-256 hashing.

Example:

```text
File: report.pdf

SHA-256:
a83f9d72b81c...
```

If another uploaded file has the same SHA-256 hash, it has identical content with an extremely high degree of confidence.

Even if the filename changes, the hash remains the same when the file content is identical.

## 3. Architecture Diagram

```text
               User Uploads File
                       |
                       v
                Amazon S3 Bucket
                       |
                       | ObjectCreated Event
                       v
                  AWS Lambda
                       |
                       v
                Read File Content
                       |
                       v
                Calculate SHA-256
                       |
                       v
                 DynamoDB Table
                       |
                Check File Hash
                       |
              +--------+--------+
              |                 |
              v                 v
        Hash Exists         Hash Not Found
              |                 |
              v                 v
        Duplicate File      New File
              |                 |
              v                 v
        Skip Processing    Save Hash in DB
                                |
                                v
                          Process File
                                |
                                v
                          CloudWatch Logs
```

## 4. How the Process Works

### Step 1: User uploads a file

Example:

```text
invoice.pdf
```

S3 receives the file and generates an `ObjectCreated` event.

### Step 2: Lambda receives the event

Lambda gets:

- Bucket name
- Object key
- File size
- Event name

### Step 3: Lambda reads the file

Lambda uses Python Boto3 to retrieve the uploaded object.

```python
response = s3.get_object(
    Bucket=bucket,
    Key=file_name
)

file_content = response["Body"].read()
```

### Step 4: Calculate SHA-256

```python
import hashlib

file_hash = hashlib.sha256(file_content).hexdigest()
```

### Step 5: Check DynamoDB

DynamoDB stores previously processed hashes.

Example:

| File name | SHA-256 | Status |
|---|---|---|
| invoice.pdf | abc123 | PROCESSED |
| image.png | xyz456 | PROCESSED |

Lambda checks whether the hash already exists.

### Step 6: Decide whether the file is duplicate

If the hash exists:

```text
Duplicate file detected.
Processing skipped.
```

If the hash does not exist:

```text
New file detected.
Processing started.
```

Lambda saves the hash in DynamoDB.

## 5. DynamoDB Table Design

Table name:

```text
FileUploadRegistry
```

Partition key:

```text
file_hash (String)
```

Attributes:

| Attribute | Description |
|---|---|
| file_hash | SHA-256 fingerprint |
| file_name | Uploaded filename |
| bucket_name | S3 bucket |
| status | PROCESSING / PROCESSED / FAILED |
| created_at | Upload processing timestamp |

Using the hash as the partition key allows us to look up content duplicates regardless of their filenames.

## 6. Python Lambda Implementation

Create:

```text
lambda_function.py
```

Example implementation:

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

        response = s3.get_object(
            Bucket=bucket,
            Key=file_name
        )

        file_content = response["Body"].read()

        file_hash = hashlib.sha256(
            file_content
        ).hexdigest()

        print("SHA-256:", file_hash)

        try:

            table.put_item(
                Item={
                    "file_hash": file_hash,
                    "file_name": file_name,
                    "bucket_name": bucket,
                    "status": "PROCESSING"
                },
                ConditionExpression=(
                    "attribute_not_exists(file_hash)"
                )
            )

            print("New file detected.")

            # Add actual file-processing logic here.

            table.update_item(
                Key={"file_hash": file_hash},
                UpdateExpression="SET #s = :status",
                ExpressionAttributeNames={
                    "#s": "status"
                },
                ExpressionAttributeValues={
                    ":status": "PROCESSED"
                }
            )

            print("File processed successfully.")

        except ClientError as error:

            if error.response["Error"]["Code"] == (
                "ConditionalCheckFailedException"
            ):

                print("Duplicate file detected.")
                print("Processing skipped.")

            else:
                raise

    return {
        "statusCode": 200,
        "body": "File upload check completed"
    }
```

**Important:** This is a basic demonstration. For production, add failure recovery, conditional status transitions, large-file streaming and protection against concurrent duplicate processing.

## 7. IAM Permissions Required

The Lambda execution role needs:

### S3 permission

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject"
  ],
  "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
}
```

### DynamoDB permission

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:PutItem",
    "dynamodb:GetItem",
    "dynamodb:UpdateItem"
  ],
  "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/FileUploadRegistry"
}
```

Also retain the CloudWatch logging permissions.

## 8. Expected Testing Results

### Test 1: Upload a new file

```bash
echo "Hello AWS" > file1.txt

aws s3 cp file1.txt s3://YOUR-BUCKET-NAME/
```

Expected logs:

```text
Uploaded file: file1.txt
SHA-256: abc123...
New file detected.
File processed successfully.
```

### Test 2: Upload identical content with another filename

```bash
cp file1.txt file2.txt

aws s3 cp file2.txt s3://YOUR-BUCKET-NAME/
```

Expected logs:

```text
Uploaded file: file2.txt
SHA-256: abc123...
Duplicate file detected.
Processing skipped.
```

### Test 3: Upload different content

```bash
echo "Different content" > file3.txt

aws s3 cp file3.txt s3://YOUR-BUCKET-NAME/
```

Expected logs:

```text
Uploaded file: file3.txt
SHA-256: different_hash...
New file detected.
File processed successfully.
```

## 9. Important Difference: Duplicate Filename vs Duplicate Content

| Situation | Result |
|---|---|
| Same filename, same content | Same content hash; duplicate |
| Different filename, same content | Same content hash; duplicate |
| Same filename, different content | Different content hash; new content |
| Different filename, different content | Different content hash; new content |

**Important:** The above design detects duplicate content for processing. It does not prevent the S3 upload itself. If a new upload uses an existing object key, S3 may overwrite the old object before Lambda runs.

If you need to prevent overwrites, use S3 versioning, conditional writes such as `If-None-Match: *` where supported, or a separate upload API that checks the destination key before accepting the upload.

## 10. Production Considerations

1. Use DynamoDB conditional writes to avoid two Lambda invocations simultaneously processing the same hash.
2. Add a processing lease or recovery mechanism for records stuck in PROCESSING.
3. Handle Lambda retries and duplicate S3 notifications.
4. Enable S3 versioning if previous object versions must be preserved.
5. For large files, calculate SHA-256 incrementally instead of loading the whole file into memory.
6. Consider SQS between S3 and Lambda for buffering and controlled processing.
7. Add lifecycle rules to manage old S3 objects and DynamoDB records.

## 11. Interview Questions

**Q1. Does S3 automatically detect duplicate files?**

No. S3 identifies objects using bucket and key. Duplicate-content detection requires additional logic.

**Q2. How do you identify identical files with different names?**

Calculate SHA-256 for the file contents and compare the hashes.

**Q3. Why use DynamoDB?**

To maintain a persistent registry of previously processed file hashes.

**Q4. Why use a conditional write?**

To ensure only one Lambda invocation can claim a previously unseen hash.

**Q5. Does ETag always equal the MD5 checksum?**

No. Multipart uploads and some encryption configurations mean ETag should not be treated as a universal content hash.

**Q6. Can Lambda stop S3 from uploading a duplicate file?**

Not after the S3 upload has already completed. Lambda is triggered after the object is created. To reject an upload before storage, use an upload API or application-side validation.

## Conclusion

This architecture provides content-based duplicate detection using:

```text
S3 + Lambda + SHA-256 + DynamoDB
```

It is useful for document processing, image uploads, backup systems, data ingestion pipelines and serverless applications.
