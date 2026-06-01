# Code Explanation — encryption_report_lambda.py

This document explains how `encryption_report_lambda.py` works, block by block.

## 1. Imports and Configuration

```python
import boto3
import openpyxl
from openpyxl.styles import Font, Alignment, Border, Side, PatternFill
from datetime import datetime
from io import BytesIO
```

- `boto3` — AWS SDK for Python (included in Lambda runtime, no layer needed)
- `openpyxl` — Excel file generation (provided via Lambda Layer)
- `datetime` — For generating year/month in the S3 path
- `BytesIO` — In-memory file buffer (no disk I/O needed)

```python
ACCOUNTS = {
    'dev': {'role_arn': 'arn:aws:iam::058264257786:role/inventory-readonly-role'},
    'uat': {'role_arn': 'arn:aws:iam::891377165721:role/inventory-readonly-role'},
    ...
}
OUTPUT_BUCKET = 'aws-artifact-collector'
```

- `ACCOUNTS` — Dictionary mapping environment names to their cross-account IAM role ARNs
- `OUTPUT_BUCKET` — S3 bucket where the final Excel report is uploaded

## 2. Region Discovery

```python
def get_all_regions(session):
    ec2_client = session.client('ec2', region_name='ap-south-1')
    regions = ec2_client.describe_regions(
        Filters=[{'Name': 'opt-in-status', 'Values': ['opt-in-not-required', 'opted-in']}]
    )
    return [region['RegionName'] for region in regions['Regions']]
```

**Purpose:** Dynamically discovers all enabled AWS regions for the account.

**Logic:**
- Uses `describe_regions` API with a filter to get only active regions
- `opt-in-not-required` = default regions (us-east-1, eu-west-1, ap-south-1, etc.)
- `opted-in` = regions the account has manually enabled (me-south-1, af-south-1, etc.)
- Returns a list of region names like `['us-east-1', 'ap-south-1', 'eu-west-1', ...]`

**Why:** Instead of hardcoding regions, this ensures we never miss resources in newly enabled regions.

## 3. Cross-Account Role Assumption

```python
def assume_role(role_arn, session_name='EncryptionAuditSession'):
    sts_client = boto3.client('sts')
    response = sts_client.assume_role(
        RoleArn=role_arn,
        RoleSessionName=session_name,
        DurationSeconds=3600
    )
    credentials = response['Credentials']
    session = boto3.Session(
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken'],
    )
    return session
```

**Purpose:** Securely access another AWS account without storing any credentials.

**Logic:**
1. Calls `sts:AssumeRole` with the target account's role ARN
2. AWS returns temporary credentials (AccessKeyId, SecretAccessKey, SessionToken)
3. These credentials are valid for 1 hour (`DurationSeconds=3600`)
4. Creates a new `boto3.Session` using these temporary credentials
5. All subsequent API calls use this session → they execute in the target account

**Security:** No long-lived access keys. Credentials auto-expire. If compromised, damage is time-limited.

## 4. EC2/EBS Encryption Collection

```python
def get_ec2_encryption_details(session, account_name, region):
```

**Purpose:** Finds all EC2 instances and checks encryption status of their attached EBS volumes.

**Logic:**
1. **Get all EC2 instances** using paginated `describe_instances`:
   ```python
   paginator = ec2_client.get_paginator('describe_instances')
   for page in paginator.paginate():
       for reservation in page['Reservations']:
           for instance in reservation['Instances']:
               instances.append(instance)
   ```

2. **Get all EBS volumes** using paginated `describe_volumes`:
   ```python
   vol_paginator = ec2_client.get_paginator('describe_volumes')
   for page in vol_paginator.paginate():
       for volume in page['Volumes']:
           volumes[volume['VolumeId']] = volume
   ```

3. **Map instances to their volumes:**
   - For each instance, extract the `Name` tag (falls back to instance ID if no name)
   - Loop through `BlockDeviceMappings` to find attached volume IDs
   - Look up each volume's `Encrypted` flag and `KmsKeyId`

4. **Resolve encryption type:**
   - If `Encrypted=True` and `KmsKeyId` exists → resolve to friendly alias (e.g., `KMS: aws/ebs`)
   - If `Encrypted=True` but no key → `Encrypted (key unknown)`
   - If `Encrypted=False` → `Not Encrypted`

**Why pagination:** AWS APIs return max 50-100 results per call. Pagination ensures we get ALL resources even in large accounts.

**Early return optimization:** If no instances exist in a region, we skip the volume lookup entirely.

## 5. S3 Encryption Collection

```python
def get_s3_encryption_details(session, account_name):
```

**Purpose:** Checks default encryption configuration for every S3 bucket in the account.

**Logic:**
1. **List all buckets** (S3 is a global service, so this is called once per account, not per region):
   ```python
   buckets = s3_client.list_buckets().get('Buckets', [])
   ```

2. **For each bucket, get encryption config:**
   ```python
   enc_response = s3_client.get_bucket_encryption(Bucket=bucket_name)
   rules = enc_response['ServerSideEncryptionConfiguration']['Rules']
   sse_algorithm = rules[0]['ApplyServerSideEncryptionByDefault']['SSEAlgorithm']
   ```

3. **Map SSE algorithm to friendly name:**
   - `AES256` → `Server-side encryption with Amazon S3 managed keys (SSE-S3)`
   - `aws:kms` → Resolves KMS key to alias (e.g., `KMS: aws/s3`)
   - `aws:kms:dsse` → `Dual-layer server-side encryption with AWS KMS keys (DSSE-KMS)`

4. **Error handling:**
   - `ServerSideEncryptionConfigurationNotFoundError` → bucket has no default encryption → `Not Encrypted`
   - Other errors are captured and reported in the output

**Note:** Since Nov 2023, AWS enables SSE-S3 by default on all new buckets. Older buckets may still show as not encrypted if they were never configured.

## 6. RDS Encryption Collection

```python
def get_rds_encryption_details(session, account_name, region):
```

**Purpose:** Checks storage encryption for all RDS database instances.

**Logic:**
1. **List all RDS instances** using paginated `describe_db_instances`:
   ```python
   paginator = rds_client.get_paginator('describe_db_instances')
   for page in paginator.paginate():
       for db_instance in page['DBInstances']:
   ```

2. **Extract encryption details:**
   - `DBInstanceIdentifier` — database name
   - `Endpoint.Address` — connection endpoint (e.g., `mydb.abc123.ap-south-1.rds.amazonaws.com`)
   - `StorageEncrypted` — boolean flag
   - `KmsKeyId` — ARN of the KMS key used

3. **Resolve encryption type:**
   - If `StorageEncrypted=True` → resolve KMS key alias (e.g., `KMS: aws/rds`)
   - If `StorageEncrypted=False` → `Not Encrypted`

**Note:** RDS encryption is set at creation time and cannot be changed later. Unencrypted databases require migration to enable encryption.

## 7. KMS Key Alias Resolution

```python
def get_kms_alias(session, kms_key_id, region):
```

**Purpose:** Converts a KMS key ARN (long, unreadable) into a friendly alias name.

**Logic:**
1. Extract the key ID from the full ARN:
   ```python
   key_id = kms_key_id.split('/')[-1]  # "arn:aws:kms:...:key/abc-123" → "abc-123"
   ```

2. Call `kms:ListAliases` for that key:
   ```python
   aliases = kms_client.list_aliases(KeyId=key_id).get('Aliases', [])
   ```

3. Return the first alias name:
   - `alias/aws/ebs` → `KMS: aws/ebs`
   - `alias/aws/rds` → `KMS: aws/rds`
   - `alias/my-custom-key` → `KMS: my-custom-key`

4. **Fallback:** If API call fails (permissions, key deleted), tries to parse alias from the ARN string itself.

**Common AWS-managed key aliases:**
- `aws/ebs` — Default EBS encryption key
- `aws/rds` — Default RDS encryption key
- `aws/s3` — Default S3 encryption key

## 8. Excel Report Generation

```python
def create_excel_report(ec2_data, s3_data, rds_data):
```

**Purpose:** Creates a formatted Excel workbook with 3 sheets.

**Logic:**
1. **Create workbook** with openpyxl:
   ```python
   wb = openpyxl.Workbook()
   ```

2. **EC2 Sheet** (first sheet, replaces default):
   - Headers: Sr No., Account Name, Resource Name, Region, Encryption Type
   - One row per EC2 instance volume

3. **S3 Sheet** (new sheet):
   - Headers: Sr No., Account Name, Resource Name, Encryption Type
   - One row per S3 bucket

4. **RDS Sheet** (new sheet):
   - Headers: Sr No., Account Name, Resource Name, Endpoint, Region, Encryption
   - One row per RDS instance

5. **Styling applied via `style_worksheet()`:**
   - Blue header row with white bold text
   - Thin borders on all cells
   - Auto-adjusted column widths (capped at 60 chars)
   - Center-aligned headers

## 9. S3 Upload

```python
def upload_to_s3(workbook):
```

**Purpose:** Saves the Excel file to S3 in the correct folder structure.

**Logic:**
1. Generate the S3 key path using current date:
   ```python
   s3_key = f'encryption/{year}/{month}/encryption_report_{year}_{month}.xlsx'
   # Example: encryption/2026/06/encryption_report_2026_06.xlsx
   ```

2. Save workbook to in-memory buffer (no temp files on disk):
   ```python
   buffer = BytesIO()
   workbook.save(buffer)
   ```

3. Upload to S3 with correct content type:
   ```python
   s3_client.put_object(
       Bucket=OUTPUT_BUCKET,
       Key=s3_key,
       Body=buffer.getvalue(),
       ContentType='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
   )
   ```

**Why BytesIO:** Lambda's `/tmp` directory is limited to 512 MB and persists between invocations. Using in-memory buffers avoids disk I/O and cleanup issues.

## 10. Lambda Handler (Entry Point)

```python
def lambda_handler(event, context):
```

**Purpose:** Main orchestrator that ties everything together.

**Execution flow:**

```
For each account in ACCOUNTS:
  │
  ├─ 1. assume_role() → Get temporary credentials
  │
  ├─ 2. get_all_regions() → Discover enabled regions
  │
  ├─ 3. get_s3_encryption_details() → S3 is global, called once
  │
  └─ 4. For each region:
       ├─ get_ec2_encryption_details(region)
       └─ get_rds_encryption_details(region)

After all accounts processed:
  │
  ├─ 5. create_excel_report() → Build Excel with all data
  │
  └─ 6. upload_to_s3() → Upload to S3
```

**Error handling:** If one account fails (bad role, permissions issue), it logs the error and continues with the next account. The report will contain data from all successful accounts.

**Return value:**
```python
{
    'statusCode': 200,
    'body': {
        'message': 'Encryption report generated successfully',
        'report_location': 's3://aws-artifact-collector/encryption/2026/06/...',
        'summary': {
            'ec2_volumes': 45,
            's3_buckets': 120,
            'rds_instances': 18
        }
    }
}
```

## 11. Cross-Account Access Flow (Visual)

```
Lambda (Central Account)
  │
  ├─ sts:AssumeRole → arn:aws:iam::111111111111:role/inventory-readonly-role (DEV)
  │     ↓
  │   Temp credentials → describe_instances, describe_volumes, list_buckets, etc.
  │
  ├─ sts:AssumeRole → arn:aws:iam::222222222222:role/inventory-readonly-role (UAT)
  │     ↓
  │   Temp credentials → describe_instances, describe_volumes, list_buckets, etc.
  │
  ├─ sts:AssumeRole → arn:aws:iam::333333333333:role/inventory-readonly-role (PROD)
  │     ↓
  │   ...
  │
  └─ s3:PutObject → s3://aws-artifact-collector/encryption/2026/06/report.xlsx
```

## 12. Key Design Decisions

| Decision | Reason |
|----------|--------|
| Pagination everywhere | AWS APIs cap results at 50-100 per call; pagination ensures completeness |
| S3 called once per account | S3 is a global service; calling per-region would return duplicates |
| EC2/RDS called per region | These are regional services; resources only visible in their region |
| KMS alias resolution | Raw key ARNs are unreadable; aliases like `aws/ebs` are meaningful |
| In-memory Excel buffer | Avoids Lambda `/tmp` disk limits and cleanup issues |
| Error isolation per account | One failed account doesn't block the entire report |
| Dynamic region discovery | Automatically picks up newly enabled regions without code changes |
| Styled Excel output | Professional-looking report ready for management/audit consumption |
