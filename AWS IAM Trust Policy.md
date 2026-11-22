# AWS IAM Trust Policy – Step-by-Step 

## 1. What is an IAM Role?

An IAM Role is an identity that AWS services or users can assume to get temporary permissions.

Example:

AWS Lambda needs permission to access S3.

Instead of storing AWS access keys inside Lambda, we create an IAM Role and attach the required permissions.

## 2. What is a Trust Policy?

A Trust Policy tells AWS:

**WHO is allowed to use this IAM Role?**

Example:

We want AWS Lambda to use an IAM Role.

So, we create a Trust Policy that trusts AWS Lambda.

```text
AWS Lambda
    |
    | Requests permission to assume role
    v
IAM Trust Policy
    |
    | Allows Lambda
    v
IAM Role
```

## 3. What is a Permissions Policy?

A Permissions Policy tells AWS:

**WHAT can the service do after assuming the role?**

For example:

- Read files from S3.
- Write logs to CloudWatch.
- Read or write records in DynamoDB.

## 4. Trust Policy vs Permissions Policy

| Trust Policy | Permissions Policy |
|---|---|
| Who can use the role? | What can they do? |
| Defines trusted identity | Defines AWS permissions |
| Uses `sts:AssumeRole` | Uses actions like `s3:GetObject` |
| Establishes trust | Grants access |

**Easy memory trick:**

- Trust Policy = WHO?
- Permissions Policy = WHAT?

---

## 5. Step-by-Step: Create a Trust Policy for Lambda

### Step 1: Open AWS IAM

1. Log in to AWS Console.
2. Search for IAM.
3. Open IAM.
4. Select Roles.
5. Click Create role.

### Step 2: Select Trusted Entity

Choose:

```text
Trusted entity type: AWS service
Use case: Lambda
```

This means we want Lambda to assume this role.

### Step 3: Create Trust Policy

AWS creates a trust relationship for Lambda.

The JSON looks like this:

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

### Step 4: Understand the JSON

| Field | Simple meaning |
|---|---|
| Version | IAM policy language version |
| Effect | Allow or Deny |
| Principal | Who is trusted |
| Service | AWS service name |
| Action | What action is allowed |
| sts:AssumeRole | Allows the service to assume the role |

Important line:

```json
"Service": "lambda.amazonaws.com"
```

This tells AWS that Lambda is trusted.

### Step 5: Attach Permissions Policy

Now attach the required permissions.

For basic Lambda CloudWatch logging, attach:

```text
AWSLambdaBasicExecutionRole
```

If Lambda needs to read S3 objects, also provide the required S3 permissions.

If Lambda needs to access DynamoDB, provide the required DynamoDB permissions.

### Step 6: Name the Role

Example:

```text
Role name: lambda-s3-execution-role
```

Click Create role.

Your IAM role is ready.

---

## 6. How Does It Work?

Example: User uploads a file to S3.

```text
                USER
                  |
                  | Upload file
                  v
             AMAZON S3
                  |
                  | S3 Event
                  v
              AWS LAMBDA
                  |
                  | Needs AWS permissions
                  v
          IAM ROLE ASSUMPTION
                  |
                  | Trust Policy allows Lambda
                  v
           IAM ROLE CREDENTIALS
                  |
                  | Permissions Policy
                  v
       +----------+-----------+
       |                      |
       v                      v
   AMAZON S3              CLOUDWATCH
   Read object             Write logs
```

Lambda uses temporary credentials provided through its execution role.

You do not need to put permanent AWS access keys inside your Lambda code.

---

## 7. Create Trust Policy Using Terraform

In DevOps, we can create IAM roles using Terraform.

Example:

```hcl
resource "aws_iam_role" "lambda_role" {
  name = "lambda-s3-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [{
      Effect = "Allow"

      Principal = {
        Service = "lambda.amazonaws.com"
      }

      Action = "sts:AssumeRole"
    }]
  })
}
```

Important:

`assume_role_policy` = Trust Policy.

It defines who can assume the IAM role.

---

## 8. Important Interview Questions

**Q1. What is a Trust Policy?**

A Trust Policy defines who is allowed to assume an IAM Role.

**Q2. Who creates the Trust Policy?**

We define it when creating an IAM Role using AWS Console, CLI, Terraform or CloudFormation.

**Q3. What is sts:AssumeRole?**

It is an AWS Security Token Service action that allows a trusted principal to assume a role and receive temporary credentials.

**Q4. What is the difference between Trust Policy and Permissions Policy?**

Trust Policy defines who can assume the role. Permissions Policy defines what actions the role can perform.

**Q5. Why does Lambda need an IAM Role?**

Lambda needs an IAM Role to securely access AWS services such as S3, DynamoDB and CloudWatch.

**Q6. Do we need AWS access keys inside Lambda?**

No. Lambda automatically receives temporary credentials through its execution role.

---

## 9. Final Revision

```text
IAM ROLE
   |
   +---- TRUST POLICY
   |        |
   |        +---- WHO can use the role?
   |        |
   |        +---- Example: Lambda
   |
   +---- PERMISSIONS POLICY
            |
            +---- WHAT can they do?
            |
            +---- Example: S3 Read
            +---- Example: DynamoDB Write
            +---- Example: CloudWatch Logs
```

### Remember These Three Points

1. Trust Policy = Who is allowed?
2. Permissions Policy = What is allowed?
3. IAM Role = The identity that provides temporary permissions.

**One-line interview answer:**

"We create an IAM Role with a trust policy that identifies the trusted service, such as Lambda, and attach permissions policies that define which AWS resources and actions the service can access."

## 10. Official AWS Documentation

- [AWS IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [AWS Role Trust Policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_update-role-trust-policy.html)
- [Lambda Execution Role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [AWS STS AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)

**End of Notes – AWS IAM Trust Policy**
