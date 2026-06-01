# AWS Multi-Account Encryption Report Lambda

Lambda function that collects encryption details (EC2/EBS, S3, RDS) from multiple AWS accounts across all regions using cross-account IAM role assumption, generates a styled Excel report, and stores it in S3.

No access keys. No secrets to rotate. Uses `sts:AssumeRole` for secure cross-account access with temporary credentials.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Central Account (Main)                         │
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────────┐  │
│  │  EventBridge │────▶│    Lambda     │────▶│   S3 Bucket     │  │
│  │  (Monthly)   │     │  Function    │     │  (Excel Report) │  │
│  └──────────────┘     └──────┬───────┘     └─────────────────┘  │
│                              │                                    │
└──────────────────────────────┼────────────────────────────────────┘
                               │ sts:AssumeRole
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │  Dev Account │     │  UAT Account │     │ Prod Account │
   │  (ReadOnly)  │     │  (ReadOnly)  │     │  (ReadOnly)  │
   └─────────────┘     └─────────────┘     └─────────────┘
```

### Execution Flow

```
1. EventBridge triggers Lambda monthly (1st of every month)
2. Lambda calls sts:AssumeRole for each target account
3. Gets temporary credentials (no stored secrets)
4. Discovers all enabled regions via describe_regions
5. Fetches EC2/EBS volumes, S3 buckets, RDS instances across all regions
6. Resolves KMS key aliases for friendly encryption names
7. Generates styled Excel with 3 sheets (EC2, S3, RDS)
8. Uploads to s3://aws-artifact-collector/encryption/YYYY/MM/report.xlsx
9. Returns summary with resource counts
```

## Setup

### Step 1: Create IAM Role in Each Target Account

Run this in each target account (dev, uat, prod, oldprod, network, sharedservice). Replace `XXXXXXXXXXXX` with your central/main account ID.

Create the trust policy file:

```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::XXXXXXXXXXXX:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create the role and attach ReadOnlyAccess:

```bash
aws iam create-role \
  --role-name inventory-readonly-role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name inventory-readonly-role \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

Repeat in each target account. The role name must be `inventory-readonly-role` in all accounts (or update `ACCOUNTS` dict in the code).

### Step 2: Create S3 Bucket

1. Go to **S3 Console** → **Create bucket**
2. Bucket name: `aws-artifact-collector`
3. Region: `ap-south-1` (or your preferred region)
4. Keep default settings → **Create bucket**

### Step 3: Create Lambda Layer (if you don't have one already)

> **Note:** If you already have an `openpyxl` Lambda Layer from a previous project, skip this step and reuse it.

Open **AWS CloudShell** and run:

```bash
mkdir -p python && \
pip install openpyxl --no-deps -t python/ --quiet && \
pip install et-xmlfile -t python/ --quiet && \
zip -r openpyxl-layer.zip python/ && \
aws lambda publish-layer-version \
  --layer-name openpyxl \
  --zip-file fileb://openpyxl-layer.zip \
  --compatible-runtimes python3.11 python3.12 \
  --region ap-south-1
```

### Step 4: Create Lambda Function

1. Go to **Lambda Console** → **Create function** → **Author from scratch**
2. Configure:

   | Setting | Value |
   |---------|-------|
   | Function name | `encryption-report-collector` |
   | Runtime | Python 3.11 or 3.12 |
   | Architecture | x86_64 |

3. Click **Create function**
4. Under **Code** tab → paste the entire contents of `encryption_report_lambda.py` → click **Deploy**
5. Under **Configuration** → **General configuration** → **Edit**:
   - Timeout: **15 min**
   - Memory: **512 MB**
6. Under **Code** tab → scroll to **Layers** → **Add a layer** → **Custom layers** → select `openpyxl` → **Add**

### Step 5: Lambda Execution Role (Inline Policy)

1. Go to **Configuration** → **Permissions** → click the **Role name**
2. **Add permissions** → **Create inline policy** → **JSON** tab:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeRoleInTargetAccounts",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::111111111111:role/inventory-readonly-role",
        "arn:aws:iam::222222222222:role/inventory-readonly-role",
        "arn:aws:iam::333333333333:role/inventory-readonly-role",
        "arn:aws:iam::444444444444:role/inventory-readonly-role",
        "arn:aws:iam::555555555555:role/inventory-readonly-role",
        "arn:aws:iam::666666666666:role/inventory-readonly-role"
      ]
    },
    {
      "Sid": "S3UploadReports",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::aws-artifact-collector/*"
    }
  ]
}
```

3. Name it `encryption-report-lambda-policy` → **Create policy**

## Invoke

### Manual (from Lambda Console)

Go to **Test** tab → Create test event with empty payload:

```json
{}
```

Click **Test**. This will process all accounts.

### Scheduled (EventBridge)

Go to **EventBridge** → **Schedules** → **Create schedule**:

| Schedule Name | Cron Expression | Target | Input |
|---------------|-----------------|--------|-------|
| encryption-report-monthly | `cron(0 2 1 * ? *)` | encryption-report-collector | `{}` |

This runs monthly on the 1st at 2:00 AM UTC.

## S3 Output

```
s3://aws-artifact-collector/
  encryption/
    2026/
      06/
        encryption_report_2026_06.xlsx
      07/
        encryption_report_2026_07.xlsx
```

Each Excel file contains 3 sheets:

| Sheet | Columns |
|-------|---------|
| EC2 | Sr No., Account Name, Resource Name, Region, Encryption Type |
| S3 | Sr No., Account Name, Resource Name, Encryption Type |
| RDS | Sr No., Account Name, Resource Name, Endpoint, Region, Encryption |



## Adding New Accounts

1. In the new target account, create the role:
   ```bash
   aws iam create-role --role-name inventory-readonly-role \
     --assume-role-policy-document file://trust-policy.json
   aws iam attach-role-policy --role-name inventory-readonly-role \
     --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
   ```

2. In `encryption_report_lambda.py`, add to `ACCOUNTS` dict:
   ```python
   'newaccount': {
       'role_arn': 'arn:aws:iam::777777777777:role/inventory-readonly-role',
   },
   ```

3. Update the Lambda inline policy to include the new role ARN in the `Resource` array.

4. Deploy the updated code.

## Project Files

```
├── encryption_report_lambda.py   # Paste into Lambda console code editor
├── README.md                     # This setup guide
├── CODE_EXPLANATION.md           # Detailed code walkthrough
└── requirements.txt              # Dependencies (for local testing only)
```

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `No module named 'openpyxl'` | Layer not attached | Add the openpyxl layer to the function |
| `AccessDenied` on AssumeRole | Trust policy missing or wrong account ID | Check trust policy in target account allows your central account |
| `AccessDenied` on PutObject | Missing S3 permission in Lambda role | Add `s3:PutObject` for the bucket in inline policy |
| `Task timed out` | Too many resources across all regions | Increase timeout to 15 min and memory to 1024 MB |
| `is not authorized to perform: sts:AssumeRole` | Lambda role missing AssumeRole permission | Add the target role ARN to the inline policy Resource array |
| Empty EC2/RDS sheets | No resources in scanned regions | Verify resources exist; check CloudWatch logs for errors |
