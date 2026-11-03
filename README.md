# Serverless DynamoDB Backup Pipeline (AWS Lambda + DynamoDB + S3 + EventBridge Scheduler)

An end-to-end serverless data pipeline built on AWS. It fetches sample users from a public REST API, stores them in DynamoDB, and exports the table to timestamped JSON files in a private S3 bucket every 2 minutes.

## Architecture

```text
           https://jsonplaceholder.typicode.com/users
                            |
                            | HTTP GET (10 sample users)
                            v
           [Lambda #1: import-jsonplaceholder-users]
                            |
                            | boto3: dynamodb:PutItem
                            v
              [DynamoDB: jsonplaceholder-users]
                            |
                            | boto3: dynamodb:Scan
                            v
         [Lambda #2: export-dynamodb-to-s3] <--- [EventBridge Scheduler]
                            |                      rate(2 minutes)
                            | boto3: s3:PutObject
                            v
                [Private Amazon S3 Bucket]
                backups/YYYY-MM-DD/users-...json

      Both Lambda functions ---> CloudWatch Logs / Metrics
```

## AWS Services Used

| Service | Role in the project |
| --- | --- |
| **AWS Lambda** (Python 3.12) | Two functions: one imports data, one exports backups |
| **Amazon DynamoDB** | Stores user records (partition key `id`, type Number) |
| **Amazon S3** | Private bucket holding timestamped JSON backups |
| **EventBridge Scheduler** | Invokes the export Lambda with `rate(2 minutes)` |
| **IAM** | Least-privilege execution roles for each function and the scheduler |
| **CloudWatch** | Logs and metrics for both functions |

## Repository Structure

```text
.
├── README.md
├── lambda_zips/
│   ├── lambda-import-users.zip        # Deployed to Lambda #1
│   └── lambda-backup-to-s3.zip        # Deployed to Lambda #2
├── lambda_src/
│   ├── import_users/lambda_function.py
│   └── backup_users/lambda_function.py
└── iam_policies/
    ├── lambda-import-putitem-policy.json
    └── lambda-backup-export-policy.json
```

## How It Works

### Lambda #1: `import-jsonplaceholder-users`
- Calls the public JSONPlaceholder API using Python's built-in `urllib`.
- Loops through the 10 returned users and writes each one with `table.put_item(Item=user)`.
- Because `PutItem` replaces items with the same `id`, re-running it does **not** create duplicates; the table stays at 10 items.
- Environment variable: `TABLE_NAME`

### Lambda #2: `export-dynamodb-to-s3`
- Scans **all pages** of the DynamoDB table (handles pagination).
- Converts DynamoDB `Decimal` values to JSON-safe numbers.
- Uploads a JSON array to S3 under a unique key built from the UTC date, time (with microseconds), and the Lambda request ID, so no backup ever overwrites another:

  ```text
  s3://<bucket>/backups/2026-11-03/users-12-30-00-123456-<REQUEST_ID>.json
  ```
- Environment variables: `TABLE_NAME`, `BUCKET_NAME`, `BACKUP_PREFIX` (`backups`)

### EventBridge Scheduler
- Schedule `dynamodb-backup-every-2min` with `rate(2 minutes)` targets Lambda #2.
- It uses its own scheduler execution role (separate from the Lambda execution role) that is allowed to invoke the function.

## IAM Permissions (Least Privilege)

| Principal | Permissions |
| --- | --- |
| Lambda #1 execution role | `dynamodb:PutItem` on the `jsonplaceholder-users` table + CloudWatch Logs |
| Lambda #2 execution role | `dynamodb:Scan` on the table, `s3:PutObject` on `backups/*` + CloudWatch Logs |
| Scheduler execution role | `lambda:InvokeFunction` on Lambda #2 |

Policy templates are in `iam_policies/`. Replace the `REGION`, `ACCOUNT_ID`, and `YOUR_UNIQUE_BUCKET_NAME` placeholders with your own values before use. No AWS credentials are stored in code; Boto3 uses the execution role.

## Setup Summary

1. **DynamoDB:** Create table `jsonplaceholder-users` with partition key `id` (**Number**).
2. **S3:** Create a private bucket (Block Public Access on, default SSE-S3 encryption).
3. **Lambda #1:** Create `import-jsonplaceholder-users`, attach the PutItem inline policy, set `TABLE_NAME`, upload `lambda-import-users.zip`, handler `lambda_function.lambda_handler`.
4. **Lambda #2:** Create `export-dynamodb-to-s3`, attach the Scan/PutObject inline policy, set `TABLE_NAME`, `BUCKET_NAME`, `BACKUP_PREFIX`, upload `lambda-backup-to-s3.zip`, same handler.
5. **Test** both functions manually with the event `{}`.
6. **Schedule:** Create the EventBridge Scheduler rule with `rate(2 minutes)` targeting Lambda #2.
7. **Observe:** Check CloudWatch logs/metrics and the `backups/` prefix in S3.

Recommended Lambda settings: Python 3.12, x86_64, 128 MB memory, 30-second timeout, not attached to a custom VPC.

## Expected Results

- Lambda #1 returns `{"statusCode": 200, "body": "{\"table\": \"jsonplaceholder-users\", \"saved_count\": 10}"}`
- Lambda #2 returns the exported count, the S3 URI, and the number of scan pages.
- With the schedule enabled, about 30 new JSON files appear per hour.

## Important Notes

- **This is an export exercise, not a production backup strategy.** A table `Scan` is not a point-in-time snapshot. It does not preserve indexes, table configuration, alarms, or access policies, and large scans can exceed Lambda limits and consume read capacity.
- For production, use **DynamoDB Point-in-Time Recovery (PITR)** and the managed **DynamoDB Export to S3** feature.
- A 2-minute schedule creates roughly 21,600 objects per 30 days. Consider an S3 lifecycle rule (for example, expire `backups/` after 7 days).

## Cost and Cleanup

This project creates billable resources. When finished:

1. Disable, then delete, the `dynamodb-backup-every-2min` schedule.
2. Delete both Lambda functions.
3. Delete the DynamoDB table.
4. Empty and delete the S3 bucket.
5. Delete the CloudWatch log groups and lab-only IAM roles/policies.

## Security

- Never commit AWS access keys, secrets, or real account IDs. Keep placeholders in the IAM policy files.
- Keep the S3 bucket private with Block Public Access enabled.

