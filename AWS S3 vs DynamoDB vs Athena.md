# AWS S3 vs DynamoDB vs Athena – Complete Beginner Guide

## 1. Introduction

Amazon S3, DynamoDB, and Athena are AWS services used for different purposes.

| AWS Service | Main Purpose |
|---|---|
| Amazon S3 | Store files and objects |
| Amazon DynamoDB | Store and query NoSQL application data |
| Amazon Athena | Run SQL queries on data stored in S3 |

**Remember:**
- S3 = Object storage
- DynamoDB = NoSQL database
- Athena = SQL queries on S3 data

---

## 2. What is Amazon S3?

Amazon S3 (Simple Storage Service) is an object storage service.

It stores data as objects inside buckets.

### What is an S3 object?

An S3 object contains:

1. Object Key – Name or path of the object.
2. Object Data – Actual file content.
3. Metadata – Information about the object.
4. Version ID – Identifies a particular version when versioning is enabled.

### Example

```text
my-company-bucket/
│
├── users/
│   ├── user1/profile.jpg
│   └── user2/profile.jpg
│
├── documents/
│   ├── invoice.pdf
│   └── resume.pdf
│
├── videos/
│   └── training.mp4
│
└── backups/
    └── database-backup.sql
```

### What data can we store in S3?

| Data | Example |
|---|---|
| Images | profile.jpg, photo.png |
| Videos | training.mp4 |
| Documents | invoice.pdf, resume.pdf |
| Text files | report.txt |
| Application files | Frontend build files |
| Backups | Database backup |
| Data science files | CSV, JSON, Parquet |
| Large files | ZIP, TAR |

**Important:** S3 stores the actual file content, not just the file name.

---

## 3. What is DynamoDB?

Amazon DynamoDB is a fully managed NoSQL database provided by AWS.

It stores data using tables, items, and attributes.

### SQL vs NoSQL

| Feature | SQL | NoSQL |
|---|---|---|
| Data structure | Tables, rows, columns | Depends on database model |
| Schema | Usually predefined | Often flexible |
| Relationships | Supports joins | No traditional relational joins |
| Querying | SQL | Database-specific APIs and query models |
| Examples | MySQL, PostgreSQL | DynamoDB, MongoDB |

NoSQL means **Not Only SQL**.

There are four common NoSQL database types:

| Type | Data Structure | Example |
|---|---|---|
| Key-value | Key and value | Redis |
| Document | JSON-like documents | MongoDB |
| Wide-column | Column families | Cassandra |
| Graph | Nodes and relationships | Neo4j |

### How DynamoDB stores data

DynamoDB supports key-value and document-style data.

Example:

```json
{
  "user_id": "101",
  "name": "David",
  "skills": [
    "AWS",
    "Docker",
    "Kubernetes"
  ],
  "address": {
    "city": "Hyderabad",
    "country": "India"
  }
}
```

Important DynamoDB concepts:

- Table – Container for items.
- Item – Individual record.
- Attribute – A field inside an item.
- Partition Key – Identifies the partition for an item.
- Sort Key – Optional key used to organize and query related items.
- GSI – Global Secondary Index.
- LSI – Local Secondary Index.

### How to query DynamoDB using Python

```python
response = table.get_item(
    Key={
        "user_id": "101"
    }
)

print(response["Item"])
```

DynamoDB also supports PartiQL, which provides SQL-compatible query syntax for supported operations.

Example:

```sql
SELECT * FROM Users
WHERE user_id = '101'
```

However, DynamoDB is not a traditional relational SQL database. Its data access is primarily designed around keys and indexes.

---

## 4. S3 Object vs DynamoDB Item

Both services store data, but their purposes are different.

| Feature | S3 | DynamoDB |
|---|---|---|
| Storage model | Object | Item |
| Actual file content | Yes | Not intended for large files |
| JSON data | Can store as a file | Native document-style attributes |
| Metadata | Object metadata | Item attributes |
| Main purpose | File storage | Application database |
| Access | Bucket + object key | Partition key + optional sort key |

### Example: Uploading a profile picture

**S3 stores the actual image:**

```text
Bucket: my-company-bucket

Object:
users/101/profile.jpg

Object data:
[Actual image bytes]
```

**DynamoDB stores information about the image:**

```json
{
  "user_id": "101",
  "name": "David",
  "image_key": "users/101/profile.jpg",
  "image_bucket": "my-company-bucket"
}
```

The DynamoDB record tells the application where the actual image is stored in S3.

---

## 5. Can We Query S3 Like DynamoDB?

Yes, but the query capabilities are different.

S3 is object storage, not a traditional database.

You can find S3 objects using:

- Bucket name
- Object key
- Prefix
- Object tags and metadata, using appropriate APIs or indexing services

### Example S3 commands

List objects:

```bash
aws s3 ls s3://my-bucket/
```

List objects under a prefix:

```bash
aws s3 ls s3://my-bucket/users/
```

Get information about an object:

```bash
aws s3api head-object \
  --bucket my-bucket \
  --key documents/invoice.pdf
```

Download an object:

```bash
aws s3 cp \
  s3://my-bucket/documents/invoice.pdf \
  ./invoice.pdf
```

**Important:** Listing an object does not mean S3 is searching inside the file's contents.

---

## 6. How to Query Data Inside S3?

AWS provides additional services for querying data stored in S3.

### A. Amazon S3 Select

S3 Select allows SQL-like queries on supported object formats, such as CSV, JSON and Parquet.

It can retrieve selected records from an individual object instead of requiring the entire object to be downloaded.

### B. Amazon Athena

Athena is a serverless interactive query service.

It allows you to run SQL queries directly against supported data stored in S3.

Example CSV file stored in S3:

```csv
name,city,salary
David,Hyderabad,100000
John,Bangalore,80000
Peter,Chennai,90000
```

You can query this data using Athena:

```sql
SELECT *
FROM employees
WHERE salary > 90000;
```

Expected result:

```text
David | Hyderabad | 100000
```

Athena uses table definitions, usually registered in the AWS Glue Data Catalog, to understand the structure of the data.

### C. AWS Glue Data Catalog

Glue Data Catalog stores metadata about datasets, such as:

- Table name
- Column names
- Data types
- S3 data location
- File format

It helps Athena understand how to query data in S3.

### S3 query services comparison

| AWS Service | Purpose |
|---|---|
| S3 API | Find and retrieve objects |
| S3 Select | SQL-like queries on an individual supported object |
| Athena | SQL queries across datasets in S3 |
| Glue Data Catalog | Stores table and schema information |
| OpenSearch | Search indexed text and documents |

---

## 7. Real-World Architecture: S3 + Lambda + DynamoDB

Let's understand a practical file upload application.

### Architecture

```text
              USER
                |
                | Upload invoice.pdf
                v
          AMAZON S3 BUCKET
                |
                | S3 ObjectCreated Event
                v
            AWS LAMBDA
                |
                | Read uploaded file
                | Calculate SHA-256
                | Extract file information
                v
           DYNAMODB TABLE
                |
                | Store file metadata
                | Store SHA-256 hash
                | Store processing status
                v
          APPLICATION
                |
                | Retrieve file information
                | Find S3 object using its key
                v
          AMAZON S3
                |
                v
          Download File
```

### What goes into S3?

```text
Bucket: david-upload-bucket

Object:
documents/invoice.pdf

Actual content:
[PDF file bytes]
```

### What goes into DynamoDB?

```json
{
  "file_hash": "a1b2c3d4...",
  "file_name": "invoice.pdf",
  "bucket": "david-upload-bucket",
  "s3_key": "documents/invoice.pdf",
  "file_size": 204800,
  "status": "PROCESSED"
}
```

### Why use both?

- S3 stores the actual file.
- Lambda processes the file.
- DynamoDB stores searchable file information.
- The application uses DynamoDB to find the S3 object.
- S3 provides the actual file when the application needs it.

**Note:** An S3 event-triggered Lambda runs after the upload. It can detect duplicate content and skip duplicate processing, but it does not prevent the upload itself.

---

## 8. When Should We Use Each AWS Service?

| Requirement | Suitable Service |
|---|---|
| Store images and videos | S3 |
| Store PDF documents | S3 |
| Store backups | S3 |
| Store user profile information | DynamoDB |
| Store shopping cart information | DynamoDB |
| Store order status | DynamoDB |
| Store file metadata | DynamoDB |
| Calculate file hashes | Lambda + Python |
| Query CSV files using SQL | Athena |
| Search indexed documents | OpenSearch |

---

## 9. Important Interview Questions and Answers

**Q1. Is Amazon S3 a database?**

No. S3 is an object storage service, not a traditional database.

**Q2. Is DynamoDB SQL or NoSQL?**

DynamoDB is a NoSQL database. It supports key-value and document-style data.

**Q3. What is an S3 object?**

An S3 object consists of object data, a key, metadata and optionally a version ID.

**Q4. What is a DynamoDB item?**

An item is an individual record in a DynamoDB table. It contains attributes.

**Q5. Can we store JSON in S3?**

Yes. JSON can be stored as a regular object in S3.

**Q6. Can we store JSON in DynamoDB?**

Yes. DynamoDB supports document-style attributes, including maps and lists.

**Q7. Can we run SQL queries on S3?**

Yes. Amazon Athena can run SQL queries on supported datasets stored in S3. S3 Select also supports SQL-like queries on supported individual objects.

**Q8. What is the difference between Athena and DynamoDB?**

Athena is designed for SQL analytics on data sources such as S3. DynamoDB is designed for low-latency application data access using keys and indexes.

**Q9. Why use DynamoDB with S3?**

S3 stores the actual files, while DynamoDB stores metadata and references to those files. This makes it easier for applications to find and manage uploaded files.

**Q10. Can DynamoDB replace S3?**

Not normally. DynamoDB has a 400 KB maximum item size. S3 is designed for large objects and files.

---

## 10. Final Revision – Remember These Three Services

```text
S3
 |
 | Stores actual files
 | PDF, JPG, MP4, CSV, JSON
 |
 v
Object Storage


DYNAMODB
 |
 | Stores application records
 | User, Order, File Metadata
 |
 v
NoSQL Database


ATHENA
 |
 | Queries data stored in S3
 | SQL SELECT, WHERE, GROUP BY
 |
 v
Serverless SQL Analytics
```

### One-line interview answer

"Amazon S3 is used for object storage, DynamoDB is a managed NoSQL database for application data, and Athena is used to run SQL analytics on data stored in S3."

---

## 11. Official AWS Documentation

- [Amazon S3 Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Amazon DynamoDB Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [Amazon Athena Documentation](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
- [Amazon S3 Select Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/selecting-content-from-objects.html)
- [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html)

**End of Notes – AWS S3 vs DynamoDB vs Athena**
