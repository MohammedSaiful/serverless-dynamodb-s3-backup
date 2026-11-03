# Hands-on Assignment: AWS Lambda + DynamoDB + Scheduled S3 Backups

**Project:** S3 / Lambda / DynamoDB / S3 Backup  
**Level:** Beginner–Intermediate | **Approach:** AWS Management Console + Python  
**Goal:** Build an end-to-end serverless data pipeline and export data into new S3 JSON backup files **every 2 minutes**.

> **Read this first:** Unzip the **outer assignment ZIP** on your computer. **Do not upload the outer ZIP to Lambda.** Inside `lambda_zips/` are **two separate, ready-to-deploy function ZIPs**. Upload the correct *inner* ZIP to the correct Lambda function.

## Your challenge

Build the following pipeline from scratch:

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
                                  |                           rate(2 minutes)
                                  | boto3: s3:PutObject
                                  v
                      [Private Amazon S3 Bucket]
                      backups/YYYY-MM-DD/users-...json

           BOTH Lambda functions ---> CloudWatch Logs / Metrics
```

**What S3 means in the project name:** S3 stores the *backup output*. You may also optionally store the deployable ZIPs in S3, but **neither Lambda function is triggered by uploading a ZIP to S3**. Lambda #1 is manually invoked; Lambda #2 is invoked by EventBridge Scheduler.

## Learning outcomes

By the end, you should be able to:

1. Create and query an Amazon DynamoDB NoSQL table.
2. Build and deploy two Python AWS Lambda functions from ZIP files.
3. Use Boto3 with an IAM *execution role* (no hardcoded credentials).
4. Understand partition keys, `PutItem`, table scans, and scan pagination.
5. Create a private S3 bucket and write timestamped JSON files to it.
6. Use EventBridge Scheduler to run a Lambda function every 2 minutes.
7. Inspect function invocations and Python log output in CloudWatch.
8. Explain how JSON exports differ from production-grade database backup and PITR.

## Files in this assignment

```text
task-s3-lambda-dynamodb-s3backup.zip
└── task-s3-lambda-dynamodb-s3backup/
    ├── README.md                          <-- You are here
    ├── lambda_zips/
    │   ├── lambda-import-users.zip        <-- Upload to Lambda #1
    │   └── lambda-backup-to-s3.zip        <-- Upload to Lambda #2
    ├── lambda_src/
    │   ├── import_users/lambda_function.py
    │   └── backup_users/lambda_function.py
    ├── iam_policies/
    │   ├── lambda-import-putitem-policy.json
    │   └── lambda-backup-export-policy.json
    
```

Each **inner** ZIP contains `lambda_function.py` at the root. The Lambda **handler** for both is `lambda_function.lambda_handler`.

---

## Prerequisites and lab safety

- AWS account with permission to create IAM roles/policies, DynamoDB tables, Lambda functions, S3 buckets, and EventBridge Scheduler schedules.
- One AWS Region for **all** Lambda functions, DynamoDB, and the S3 destination (example: `us-east-1`).
- A browser and this unzipped assignment folder. **AWS CLI is not required.**
- Never put AWS access keys, secrets, or personal data in code, logs, screenshots, or submissions.
- For this public REST API exercise, leave Lambda **outside a custom VPC**. Internet access may fail if you put it in a private subnet without proper NAT routing.
- This exercise creates billable AWS resources. **Disable the 2-minute schedule and delete lab resources when finished.**
- Keep the S3 bucket private and the default S3 encryption enabled.

**Suggested settings:** Python 3.12 or another currently available supported Python runtime, 128 MB Lambda memory, 30-second timeout. These files use only `urllib` (standard Python library) and Boto3 (included in managed Lambda Python runtimes). For production, package a pinned Boto3 dependency.

---

# PART A — Set up DynamoDB and S3

## Step 1 — Select an AWS Region

1. Sign in to the AWS Management Console.
2. In the upper-right Region selector, choose one Region (for example **US East (N. Virginia) — us-east-1**).
3. Use the **same Region** for DynamoDB, both Lambdas, the S3 bucket and EventBridge Scheduler.
4. Write your choices here:

| Configuration | Your value |
| --- | --- |
| AWS Region | `________________________` |
| AWS account ID (12 digits) | `________________________` |
| DynamoDB table | `jsonplaceholder-users` |
| S3 bucket (globally unique) | `________________________` |

> To find your account ID, open your account menu in the top-right of the AWS console. Do not share account IDs publicly in screenshots unless necessary.

## Step 2 — Create the DynamoDB table

1. Open **DynamoDB → Tables → Create table**.
2. **Table name:** `jsonplaceholder-users`.
3. **Partition key:** `id`.
4. **Partition key type:** **Number** (NOT String).
5. **Sort key:** Leave blank.
6. **Table settings:** Use default settings or customize capacity to **On-demand** if the option is shown.
7. Click **Create table**; wait until the table is **Active**.
8. Open the table details and note the actual **table ARN**. It looks like:

   ```text
   arn:aws:dynamodb:REGION:ACCOUNT_ID:table/jsonplaceholder-users
   ```

**Why `id`?** It identifies each user. The other fields are flexible: `name`, `email`, `phone`, `address`, `company`, etc. DynamoDB stores nested JSON-style data too.

**Checkpoint A:** Table `jsonplaceholder-users` is Active with partition key `id` (Number).

## Step 3 — Create the destination S3 bucket

1. Open **S3 → Buckets → Create bucket**.
2. Choose a **General purpose bucket**.
3. Enter a globally unique name, e.g. `student-lambda-backups-<unique-suffix>`.
4. Select the same Region as the table/Lambdas.
5. Keep **Block all public access** enabled.
6. Keep default server-side encryption enabled (SSE-S3 is suitable for the lab).
7. Click **Create bucket**.
8. Write down the exact bucket name; you'll use it as the `BUCKET_NAME` environment variable.

You **do not** need to create a physical `backups/` folder. S3 prefixes appear automatically when objects are uploaded.

**Checkpoint B:** A private S3 bucket exists in your chosen Region.

---

# PART B — Lambda #1: Fetch API users and save them to DynamoDB

## Step 4 — Create Lambda #1

1. Open **Lambda → Functions → Create function**.
2. Choose **Author from scratch**.
3. **Function name:** `import-jsonplaceholder-users`.
4. **Runtime:** Python 3.12 or newer supported Python version.
5. **Architecture:** x86_64.
6. **Execution role:** **Create a new role with basic Lambda permissions**.
7. Click **Create function**.
8. In **Configuration → General configuration → Edit**, set timeout to **30 seconds** and memory to **128 MB**; save.

The automatically attached `AWSLambdaBasicExecutionRole` managed policy lets Lambda send logs to CloudWatch. It **does not** give DynamoDB access.

## Step 5 — Give Lambda #1 DynamoDB write permission

1. Open your first Lambda, then **Configuration → Permissions**.
2. Click the linked **Execution role** name to open IAM.
3. In IAM, select **Add permissions → Create inline policy → JSON**.
4. Open `iam_policies/lambda-import-putitem-policy.json` from this assignment.
5. Replace `REGION` and `ACCOUNT_ID` with your real values. The complete resource must match your table ARN.
6. Paste the edited JSON into IAM, click **Next**, name it `AllowPutDemoUsers`, and create the policy.
7. Keep the existing `AWSLambdaBasicExecutionRole` policy attached as well.

**Permission needed:** `dynamodb:PutItem` on *only* your demo table.

> Student question: Why isn't `AdministratorAccess` a good substitute? (Hint: least privilege.)

## Step 6 — Set environment variable and upload ZIP #1

1. Open **Lambda #1 → Configuration → Environment variables → Edit**.
2. Add `TABLE_NAME` = `jsonplaceholder-users` and **Save**.
3. Open the **Code** tab. Under **Code source**, select **Upload from → .zip file**.
4. Choose **`lambda_zips/lambda-import-users.zip`** (the *inner* ZIP, not the outer assignment ZIP).
5. Click **Save**. If the editor requires a separate **Deploy** step for changes, deploy them.
6. In **Runtime settings**, verify:

   ```text
   lambda_function.lambda_handler
   ```

The code calls the public API using Python's `urllib.request`, loops through the returned user list, and writes each full user object to DynamoDB using `table.put_item(Item=user)`.

## Step 7 — Test Lambda #1

1. Find the **Test** tab or the **TEST EVENTS** section of the Lambda editor.
2. Create a private test event named `ImportUsersTest`.
3. Paste the event JSON below (or use `sample_events/empty_test_event.json`):

   ```json
   {}
   ```

4. **Save** the event and click **Test** / **Run**.
5. Expand **Execution result / Output**.
6. On success, expect a response similar to:

   ```json
   {
     "statusCode": 200,
     "body": "{\"table\": \"jsonplaceholder-users\", \"saved_count\": 10}"
   }
   ```

7. Select **Monitor → View CloudWatch logs**, or open **CloudWatch → Logs → Log groups**.
8. Find `/aws/lambda/import-jsonplaceholder-users` and open the newest log stream.
9. Look for log lines such as:

   ```text
   Fetching users from https://jsonplaceholder.typicode.com/users
   Saved user 1: Leanne Graham
   Saved user 2: Ervin Howell
   ...
   SUCCESS: Saved 10 users to DynamoDB table jsonplaceholder-users
   ```

**If API fetch fails:** Verify public internet access and try opening the URL from a browser. If DynamoDB access is denied, recheck the Lambda #1 IAM role and table ARN.

## Step 8 — Inspect the data in DynamoDB

1. Open **DynamoDB → Tables → jsonplaceholder-users**.
2. Go to **Explore table items** / **Explore items**.
3. Choose the table and **Scan / Run** to display its records.
4. Check that item count is **10** if the public API returned its expected demo data.
5. Expand a record and find `id`, `name`, `email`, `address`, and `company`.
6. Run Lambda #1 a second time. **Why is the table still approximately 10 items?**

**Answer:** `PutItem` replaces an existing item with the same partition-key value. These records are not appended as new users on every run.

**Checkpoint C:** The table contains expected demo users and CloudWatch has log messages.

---

# PART C — Lambda #2: Export DynamoDB data into S3

## Step 9 — Create Lambda #2

1. Open **Lambda → Functions → Create function**.
2. Choose **Author from scratch**.
3. **Function name:** `export-dynamodb-to-s3`.
4. **Runtime:** Python 3.12 or newer supported Python version.
5. **Execution role:** **Create a new role with basic Lambda permissions**.
6. Click **Create function**.
7. Set memory to **128 MB** and timeout to **30 seconds** in **Configuration → General configuration**.

## Step 10 — Give Lambda #2 read and write permissions

1. Open **Lambda #2 → Configuration → Permissions → Execution role**.
2. In IAM, choose **Add permissions → Create inline policy → JSON**.
3. Open `iam_policies/lambda-backup-export-policy.json`.
4. Replace:
   - `REGION` with your DynamoDB Region.
   - `ACCOUNT_ID` with your AWS account ID.
   - `YOUR_UNIQUE_BUCKET_NAME` with your actual S3 bucket name.
5. Paste the edited policy into IAM, click **Next**, name it `AllowDynamoDBJsonBackup`, and create the policy.
6. Keep the role's `AWSLambdaBasicExecutionRole` logging policy.

This grants **only**:

- `dynamodb:Scan` on the `jsonplaceholder-users` table.
- `s3:PutObject` under `s3://YOUR_UNIQUE_BUCKET_NAME/backups/`.

The export Lambda does **not** require access to delete S3 objects or change DynamoDB data.

## Step 11 — Set environment variables and upload ZIP #2

In **Lambda #2 → Configuration → Environment variables**, add:

| Environment variable | Value |
| --- | --- |
| `TABLE_NAME` | `jsonplaceholder-users` |
| `BUCKET_NAME` | Your **actual** private S3 bucket name |
| `BACKUP_PREFIX` | `backups` |

Then:

1. Open the **Code** tab and choose **Upload from → .zip file**.
2. Upload **`lambda_zips/lambda-backup-to-s3.zip`**.
3. Save/deploy changes as required.
4. Verify handler is `lambda_function.lambda_handler`.

The code **scans all pages** of the DynamoDB table, converts DynamoDB `Decimal` numbers to JSON-safe values, and uploads a JSON array to S3.

Every run creates a **new unique object key** using the UTC date, time (including microseconds), and the Lambda request ID:

```text
s3://YOUR_UNIQUE_BUCKET_NAME/backups/2026-10-23/users-12-30-00-123456-REQUEST_ID.json
```

The time above is only an example. **Each run keeps its own JSON export instead of overwriting the previous one.**

## Step 12 — Manually test Lambda #2

1. Open Lambda #2's **Test** tab / **TEST EVENTS** panel.
2. Create an event named `ExportUsersTest` using `{}`.
3. Save and click **Test / Run**.
4. Look for a response similar to:

   ```json
   {
     "statusCode": 200,
     "body": "{\"exported_count\": 10, \"s3_uri\": \"s3://YOUR_UNIQUE_BUCKET_NAME/backups/2026-10-23/users-...json\", \"scan_pages\": 1}"
   }
   ```

5. Open **S3 → Your bucket → backups/ → (today's UTC date)**.
6. Confirm there is a new `users-...json` file.
7. Download and inspect it. It should be a list of user objects, including names, emails, and nested address/company information.
8. Run the Lambda a **second time** and verify a **second JSON file** appears.
9. Inspect CloudWatch logs for `/aws/lambda/export-dynamodb-to-s3`.

**Checkpoint D:** At least two separate timestamped JSON exports exist in S3.

---

# PART D — Automate a new S3 export every 2 minutes

## Step 13 — Create the EventBridge Scheduler schedule

**Recommended: use the EventBridge Scheduler console.**

1. Open **Amazon EventBridge Scheduler** (`https://console.aws.amazon.com/scheduler/home`).
2. Click **Create schedule**.
3. **Name:** `dynamodb-backup-every-2min`.
4. **Schedule group:** `default`.
5. **Schedule type:** **Recurring**.
6. **Frequency:** **Rate-based**.
7. **Rate expression:** `rate(2 minutes)` (in form UIs: Value `2`, Unit `minutes`).
8. **Flexible time window:** **Off**.
9. Set schedule to **Enabled**; optionally specify a start and end date/time if the UI allows. Check that the start time is in the future if one is set.
10. Under **Target**, choose **AWS Lambda Invoke** (or AWS Lambda as target).
11. Choose the **`export-dynamodb-to-s3`** Lambda function in the **same Region**.
12. **Payload / Input:** `{}` or leave empty if the UI allows.
13. **Scheduler execution role:** choose **Create new role for this schedule** so Scheduler receives permission to invoke the target function. This role is **different** from the Lambda #2 execution role.
14. Review and **Create schedule**.

> Alternative: In **Lambda #2 → Function overview → Add trigger**, choose **Scheduler**, then create a recurring schedule. The exact screen layout may differ from the standalone Scheduler console.

**Important scheduling detail:** A rate of 2 minutes is approximately one invocation every 2 minutes, not a promise of execution at an exact second. With flexible time window Off, Scheduler still has minute-level delivery precision; retries or errors can affect actual file counts.

## Step 14 — Verify the automatic runs

1. In **EventBridge Scheduler → Schedules**, verify `dynamodb-backup-every-2min` is **Enabled**.
2. Open **CloudWatch → Log groups → `/aws/lambda/export-dynamodb-to-s3`**.
3. Watch for new log streams / requests as Scheduler triggers the function.
4. Open S3 → Your bucket → `backups/YYYY-MM-DD/`.
5. Confirm multiple **distinct** `users-...json` files appear over successive successful runs.
6. Open **Lambda #2 → Monitor → Metrics** and find `Invocations`, `Errors`, and `Duration`.

**Expected frequency if all runs succeed:** ~30 files/hour, ~720/day, ~21,600/30 days. Even this small lab can accumulate S3 objects and AWS charges. **Disable the schedule once you have proved it works.**

**Checkpoint E:** Scheduler is enabled, two or more *automatic* invocations are visible, and the corresponding new files exist in S3.

---

# PART E — Observe, troubleshoot, and clean up

## Step 15 — Observe CloudWatch metrics and logs

For **each** Lambda function:

1. Open **Lambda → Function → Monitor → Metrics**.
2. Inspect `Invocations`, `Errors`, `Duration`, and (if shown) `Throttles`.
3. Open **Monitor → View CloudWatch logs**.
4. Find logs beginning with the Python messages from the supplied source.

**Explain the difference:**

- **Metrics** measure how often code runs, how long it takes, and whether the invocation failed.
- **Logs** show detailed application messages (`Saved user...`, `Exported 10 users...`) and runtime `START` / `END` / `REPORT` information.
- The names and number of users are **application logs**, not automatic custom CloudWatch metrics.

## Step 16 — Optional: keep backups for only 7 days

1. Open **S3 → Your bucket → Management → Create lifecycle rule**.
2. Name the rule `expire-demo-backups-after-7-days`.
3. Limit scope to **Prefix**: `backups/`.
4. Choose **Expire current versions of objects** after **7 days**.
5. Review and create the rule.

If **S3 Versioning** is enabled, configure expiration of **noncurrent versions** separately; otherwise old versions may still be billed. Lifecycle actions are asynchronous and may not execute exactly at the expiration boundary.

## Step 17 — Common problems and fixes

| Symptom / error | Likely cause | Fix |
| --- | --- | --- |
| `KeyError: 'TABLE_NAME'` | Missing environment variable | Add `TABLE_NAME=jsonplaceholder-users` to the correct Lambda |
| `KeyError: 'BUCKET_NAME'` | Missing backup destination | Set `BUCKET_NAME` on Lambda #2 |
| `AccessDeniedException` on `PutItem` | Import Lambda's IAM policy missing/incorrect | Add `dynamodb:PutItem` for correct table ARN |
| `AccessDeniedException` on `Scan` | Export Lambda IAM policy missing/incorrect | Add `dynamodb:Scan` for correct table ARN |
| S3 `AccessDenied` | Wrong bucket/prefix ARN or restrictive bucket policy | Ensure `s3:PutObject` covers `backups/*` |
| `ResourceNotFoundException` | Wrong table name, key, or Region | Verify exact name and same Region |
| `NoSuchBucket` | Wrong S3 bucket name | Fix `BUCKET_NAME` |
| `ValidationException` on primary key | DynamoDB `id` was made String | Recreate demo table with `id` type Number |
| `Unable to import module 'lambda_function'` | Wrong uploaded ZIP or handler | Upload the **inner** function ZIP; handler must be `lambda_function.lambda_handler` |
| Function times out calling the API | No internet access, API outage, or short timeout | Check networking; for demo don't attach Lambda to a private VPC; set 30-second timeout |
| No CloudWatch logs | IAM basic logging policy missing, no invocation, or wrong Region | Verify `AWSLambdaBasicExecutionRole`, invoke, allow for delivery delay |
| Scheduler created but no files | Wrong target, schedule disabled, bad Scheduler role, or Lambda error | Inspect schedule status, target, and Lambda #2 logs |
| Export has 0 users | Import Lambda never populated table or wrong table | Execute Lambda #1 and inspect DynamoDB items |
| Repeating import still shows 10 users | Expected `PutItem` behavior | Identical `id` replaces previous item |
| Export files are empty/partial | Read permissions, in-progress table writes, or scan limitation | Check logs; recognize `Scan` is not a point-in-time snapshot |

## Step 18 — Student submission checklist

Submit a short report (screenshots **without exposing credentials**) showing:

- [ ] **Architecture:** Explain the API → Lambda #1 → DynamoDB → Lambda #2 → S3 flow.
- [ ] **DynamoDB:** Table name and `id` Number partition key.
- [ ] **Lambda #1:** Successful test output, CloudWatch names, and records in DynamoDB.
- [ ] **Lambda #2:** Successful manual test and timestamped S3 JSON export.
- [ ] **IAM:** Explain why Lambda #1 needs `PutItem` while Lambda #2 needs `Scan` and `PutObject`.
- [ ] **EventBridge Scheduler:** Enabled schedule with `rate(2 minutes)`.
- [ ] **Automated proof:** Two or more S3 JSON files created from scheduled runs.
- [ ] **Observability:** CloudWatch logs and Lambda invocation graph.
- [ ] **Cleanup:** Schedule disabled or deleted after demo.

### Questions to answer in your report

1. What is the difference between **Lambda's execution role** and **Scheduler's execution role**?
2. Why does running Lambda #1 twice **not** produce 20 DynamoDB users?
3. Why do Lambda #2 backups have unique timestamps and request IDs?
4. What is the difference between `print()` logs and `Invocations`/`Errors` metrics?
5. What would happen if you deleted the DynamoDB table but kept its JSON files in S3? Could the original indexes/configuration be automatically restored from the JSON alone?
6. Why is an export every 2 minutes usually a **poor production backup strategy** for large DynamoDB tables?

## Important: This is an export exercise, not a production backup strategy

This lab makes a **JSON copy of all records read by a table `Scan`**. It is **not** a transactionally consistent, point-in-time backup; it does not preserve complete table configuration, indexes, alarms, or access policies. A scan is eventually consistent by default and cannot guarantee one snapshot across a changing table even when consistent reads are requested. Large scans may exceed Lambda's time and memory limits, and they consume DynamoDB read capacity.

For production disaster recovery, consider **DynamoDB Point-in-Time Recovery (PITR)** and managed **DynamoDB Export to S3** instead. Choose backup interval, retention, encryption, access control, and restore testing based on the workload's RPO/RTO objectives.

### Cost and security notes

- You may incur DynamoDB on-demand read/write charges, Lambda usage, S3 storage/object requests, EventBridge Scheduler invocations, and CloudWatch log charges. AWS free-tier eligibility and pricing vary.
- Do **not** turn off S3 Block Public Access or grant `s3:*` just to make the lab work.
- Do **not** embed AWS access keys in Python; Boto3 uses the function's execution-role credentials.
- In this basic lab, the S3 bucket uses default SSE-S3 encryption. For customer-managed KMS encryption, additional permissions and configuration may be needed.
- 2-minute schedules can create thousands of objects per month; create an S3 lifecycle rule and stop the schedule after class.

## Step 19 — Clean up to avoid future charges

**Do this after documenting your results:**

1. **Disable and delete** `dynamodb-backup-every-2min` in EventBridge Scheduler. Disable first so there are no further exports.
2. Delete Lambda functions `import-jsonplaceholder-users` and `export-dynamodb-to-s3`.
3. Delete the `jsonplaceholder-users` DynamoDB demo table **after checking that it contains no data you need**.
4. Delete the S3 objects in your lab bucket, then delete the bucket. If you enabled versioning, remove versions/delete markers as required.
5. Remove the two demo CloudWatch log groups if no longer needed (they can remain after deleting Lambda functions).
6. Remove any *lab-only* IAM execution roles and inline policies if no longer used by anything else.
7. Verify no Scheduler schedules, S3 objects, or DynamoDB resources remain for this lab.

---

## Official AWS references

- [AWS Lambda Python ZIP deployment](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html)
- [Writing AWS Lambda functions in Python](https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html)
- [EventBridge Scheduler: Invoke Lambda on a schedule](https://docs.aws.amazon.com/lambda/latest/dg/with-eventbridge-scheduler.html)
- [EventBridge Scheduler: Getting started](https://docs.aws.amazon.com/scheduler/latest/UserGuide/getting-started.html)
- [DynamoDB Scan and pagination (Boto3)](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb/table/scan.html)
- [AWS Lambda permissions](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
- [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/)

**Well done when you can demonstrate:** one API request → 10 stored users → a successful manual export → repeatedly scheduled S3 JSON copies → proof in CloudWatch.
