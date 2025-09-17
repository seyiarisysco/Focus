# Do you want me to also include a multi-account (AWS Organizations) setup guide so you can centralize all account exports into one S3 + Athena?

## Great 👍 Let’s extend the FOCUS implementation for multi-account (AWS Organizations) so you can centralize all AWS billing data into one place.

### Multi-Account FOCUS Setup in AWS Organizations
### Step 1: Choose a Central Billing Account

### Pick the Management (payer) account in AWS Organizations to collect all FOCUS exports.

### All member accounts will consolidate usage & cost data into this central account.

### Step 2: Create Central S3 Bucket (in Management Account)

### Terraform example:
```hcl
resource "aws_s3_bucket" "focus_org_export" {
  bucket = "org-focus-data-export-central"
}

resource "aws_s3_bucket_policy" "focus_org_policy" {
  bucket = aws_s3_bucket.focus_org_export.id
  policy = <<EOT
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBillingDelivery",
      "Effect": "Allow",
      "Principal": {
        "Service": "billingreports.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "${aws_s3_bucket.focus_org_export.arn}/*"
    }
  ]
}
EOT
}
```


## 👉 This allows AWS Billing service to deliver CUR/FOCUS exports into this bucket.

### Step 3: Enable FOCUS Export in Management Account

### Go to Billing Console → Data Exports.

### Create a new export:

### Schema = FOCUS 1.0

### Destination = s3://org-focus-data-export-central

### Format = Parquet

### ✅ This ensures all linked accounts’ usage flows into one export.


## Step 4: Partition Data by Account in Athena

Athena table example:
```sql 
CREATE EXTERNAL TABLE IF NOT EXISTS focus_org_billing (
  lineItemId string,
  payerAccountId string,
  usageAccountId string,
  serviceCode string,
  usageType string,
  operation string,
  availabilityZone string,
  region string,
  listCost double,
  billedCost double,
  effectiveCost double,
  usageAmount double,
  unit string,
  resourceId string,
  year int,
  month int
)
PARTITIONED BY (year int, month int)
STORED AS PARQUET
LOCATION 's3://org-focus-data-export-central/focus/';
```


## 👉 Run:

```sql
MSCK REPAIR TABLE focus_org_billing;
```


### to load partitions.

## Step 5: Sample Multi-Account Queries 
## Spend by Linked Account

```sql 
SELECT usageAccountId,
       SUM(effectiveCost) AS total_cost
FROM focus_org_billing
WHERE year = 2025 AND month = 8
GROUP BY usageAccountId
ORDER BY total_cost DESC;
```

## Compare Payer vs Linked Account Costs

```sql
SELECT payerAccountId,
       usageAccountId,
       SUM(effectiveCost) AS effective_cost
FROM focus_org_billing
WHERE year = 2025 AND month = 8
GROUP BY payerAccountId, usageAccountId
ORDER BY effective_cost DESC;
```

## Compare Payer vs Linked Account Costs
```sql 
SELECT payerAccountId,
       usageAccountId,
       SUM(effectiveCost) AS effective_cost
FROM focus_org_billing
WHERE year = 2025 AND month = 8
GROUP BY payerAccountId, usageAccountId
ORDER BY effective_cost DESC;
```


## Cost Breakdown by OU (if accounts are mapped in tags/metadata)
```sql
SELECT resourceTags["aws:organization:ou"],
       SUM(effectiveCost) AS ou_cost
FROM focus_org_billing
WHERE year = 2025 AND month = 8
GROUP BY resourceTags["aws:organization:ou"]
ORDER BY ou_cost DESC;
```


## Step 6: Deploy FOCUS Dashboard (Organization View)

## Use the AWS Cloud Intelligence Dashboards (CID) deployment:
```bash
aws cloudformation create-stack \
  --stack-name focus-org-dashboard \
  --template-url https://wellarchitectedlabs.s3.amazonaws.com/Cost/200_FinOps/Deployment/FOCUS/FOCUS-Dashboard.yaml \
  --parameters ParameterKey=DataSourceName,ParameterValue=focus_org_billing \
  --capabilities CAPABILITY_NAMED_IAM
```



### This creates a QuickSight dashboard with organization-wide spend.

### You’ll be able to drill down by account, OU, service, region

## ✅ End Result

### Centralized S3 bucket → collects all accounts’ cost & usage data in FOCUS format.

### Athena tables → query and analyze costs across accounts.

### QuickSight dashboards → visualize costs per service, OU, account.

# 👉 Do you also want me to include automation for OU-to-Account mapping (so you can group accounts by department automatically in FOCUS queries)?

# Perfect 🚀 Let’s extend your FOCUS multi-account setup with automated OU-to-Account mapping. This lets you see spend grouped by departments or business units (OUs) instead of just raw account IDs.

## 🔹 Step 7: Automating OU → Account Mapping in AWS Organizations

## Step 7.1: Export OU Structure from AWS Organizations

### We can use AWS CLI or boto3 to list accounts and their OU mappings.

### Example with AWS CLI:
```bash
# List all OUs under the root
aws organizations list-organizational-units-for-parent \
  --parent-id r-examplerootid

# List accounts in an OU
aws organizations list-accounts-for-parent \
  --parent-id ou-examplerootid-exampleouid
```



## Step 7.2: Automate OU-to-Account Mapping (Python Script)
```python 
import boto3
import csv

org_client = boto3.client('organizations')

def list_accounts_for_ou(ou_id):
    accounts = []
    paginator = org_client.get_paginator('list_accounts_for_parent')
    for page in paginator.paginate(ParentId=ou_id):
        for acct in page['Accounts']:
            accounts.append((acct['Id'], acct['Name'], ou_id))
    return accounts

def list_ou_structure(root_id):
    mapping = []
    paginator = org_client.get_paginator('list_organizational_units_for_parent')
    for page in paginator.paginate(ParentId=root_id):
        for ou in page['OrganizationalUnits']:
            mapping.extend(list_accounts_for_ou(ou['Id']))
    return mapping

if __name__ == "__main__":
    # Replace with your AWS Org Root ID
    root_id = "r-examplerootid"
    account_mapping = list_ou_structure(root_id)

    with open("account_ou_mapping.csv", "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["AccountId", "AccountName", "OU"])
        writer.writerows(account_mapping)

    print("✅ account_ou_mapping.csv generated!")
```

### This will generate a file like:
```
AccountId,AccountName,OU
111111111111,DevAccount,OU-Dev
222222222222,ProdAccount,OU-Prod
333333333333,Analytics,OU-Data
```


## Step 7.3: Load Mapping into Athena (Glue Table)

### Upload account_ou_mapping.csv into S3 and create a Glue table:
```sql
CREATE EXTERNAL TABLE account_ou_mapping (
  AccountId string,
  AccountName string,
  OU string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://org-focus-data-export-central/mappings/';
```


## Step 7.4: Join OU Mapping with FOCUS Data

### Now you can query costs per OU:

```sql 
SELECT m.OU,
       SUM(f.effectiveCost) AS total_cost
FROM focus_org_billing f
JOIN account_ou_mapping m
  ON f.usageAccountId = m.AccountId
WHERE year = 2025 AND month = 8
GROUP BY m.OU
ORDER BY total_cost DESC;
```

### 👉 This automatically rolls up spend by OU (like Engineering, Finance, R&D) instead of account IDs.

## Step 7.5: Automate Refresh

### Schedule the Python script (Step 7.2) via AWS Lambda + EventBridge (e.g., daily/weekly).

### Replace the CSV in S3 each run.

### Athena queries will always have the latest OU mapping.

## ✅ End Result

### FOCUS Data → Centralized in S3 (all accounts).

### OU Mapping → Automatically updated from AWS Organizations.

### Athena Queries → Show spend by OU, Account, Service, Region.

### QuickSight Dashboards → Business-friendly reporting by department.

# 👉 Do you also want me to add a CloudFormation/Terraform template that sets up the OU mapping pipeline automatically (Lambda + S3 + Glue + Athena) so you don’t need to run the script manually?

# Awesome 💡 — let’s fully automate the OU → Account mapping pipeline for your FOCUS setup using CloudFormation and Terraform. This will ensure OU mappings are always up-to-date in S3 + Athena without manual scripts.

## 🔹 CloudFormation: OU → Account Mapping Automation

### This stack does 3 things:

## Lambda Function → queries AWS Organizations and generates OU mapping CSV.

## S3 Bucket → stores account_ou_mapping.csv.

## EventBridge Rule → triggers Lambda daily (can be hourly/weekly).
```yaml 
AWSTemplateFormatVersion: '2010-09-09'
Description: "Automate OU-to-Account Mapping for FOCUS"

Resources:
  MappingBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "focus-ou-mapping-${AWS::AccountId}"

  MappingLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
        - arn:aws:iam::aws:policy/AWSOrganizationsReadOnlyAccess
        - arn:aws:iam::aws:policy/AmazonS3FullAccess

  MappingLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: OUAccountMapping
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt MappingLambdaRole.Arn
      Timeout: 120
      Code:
        ZipFile: |
          import boto3, csv, io, os
          s3 = boto3.client('s3')
          org = boto3.client('organizations')
          bucket = os.environ['BUCKET']

          def list_accounts_for_parent(pid):
              accts=[]
              paginator = org.get_paginator('list_accounts_for_parent')
              for page in paginator.paginate(ParentId=pid):
                  for acct in page['Accounts']:
                      accts.append((acct['Id'], acct['Name'], pid))
              return accts

          def handler(event, context):
              root_id = org.list_roots()['Roots'][0]['Id']
              mapping=[]
              paginator = org.get_paginator('list_organizational_units_for_parent')
              for page in paginator.paginate(ParentId=root_id):
                  for ou in page['OrganizationalUnits']:
                      mapping.extend(list_accounts_for_parent(ou['Id']))

              out = io.StringIO()
              writer = csv.writer(out)
              writer.writerow(["AccountId","AccountName","OU"])
              writer.writerows(mapping)

              s3.put_object(Bucket=bucket, Key="account_ou_mapping.csv", Body=out.getvalue())
              return {"status":"done","count":len(mapping)}

      Environment:
        Variables:
          BUCKET: !Ref MappingBucket

  MappingSchedule:
    Type: AWS::Events::Rule
    Properties:
      ScheduleExpression: rate(1 day)
      Targets:
        - Arn: !GetAtt MappingLambda.Arn
          Id: "MappingLambdaTarget"

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref MappingLambda
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt MappingSchedule.Arn
```

### 👉 After this stack is deployed:

### Every day, the Lambda will regenerate account_ou_mapping.csv in S3.

### Athena can query it directly.

## 🔹 Terraform: OU → Account Mapping Automation
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "ou_mapping" {
  bucket = "focus-ou-mapping-example"
}

resource "aws_iam_role" "lambda_role" {
  name = "focus-ou-mapping-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "orgs_access" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSOrganizationsReadOnlyAccess"
}

resource "aws_iam_role_policy_attachment" "lambda_logging" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_lambda_function" "ou_mapping" {
  function_name = "focus-ou-mapping"
  runtime       = "python3.12"
  handler       = "index.handler"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 120

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      BUCKET = aws_s3_bucket.ou_mapping.bucket
    }
  }
}

resource "aws_cloudwatch_event_rule" "daily" {
  name                = "daily-ou-mapping"
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "lambda_target" {
  rule      = aws_cloudwatch_event_rule.daily.name
  target_id = "OUMapLambda"
  arn       = aws_lambda_function.ou_mapping.arn
}

resource "aws_lambda_permission" "allow_events" {
  statement_id  = "AllowExecutionFromEvents"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ou_mapping.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily.arn
}
```
## 👉 In Terraform:

### Put the Python script (index.py) into lambda.zip.

### Deploy → Lambda runs daily → updates S3 mapping file.

## 🔹 Athena Integration

### Create a table to query the mapping:

```sql 
CREATE EXTERNAL TABLE account_ou_mapping (
  AccountId string,
  AccountName string,
  OU string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://focus-ou-mapping-example/';
```
### Join with your FOCUS billing table:

```sql
SELECT m.OU,
       SUM(f.effectiveCost) AS total_cost
FROM focus_org_billing f
JOIN account_ou_mapping m
  ON f.usageAccountId = m.AccountId
WHERE year = 2025 AND month = 8
GROUP BY m.OU
ORDER BY total_cost DESC;
```



### ✅ Now you have:

### CloudFormation/Terraform → automatic OU mapping pipeline.

### S3 → stores account_ou_mapping.csv (daily refreshed).

### Athena → query OU-level costs.

### QuickSight → visualize spend by OU, account, service.


## Would you like me to also add a QuickSight dashboard JSON template (pre-built visuals for OU spend, account spend, discounts, service breakdown) so you can deploy a FinOps OU dashboard in one click?

## Perfect 🚀 — let’s finish your setup with a QuickSight OU Dashboard template that plugs into your FOCUS Athena tables and OU mapping.

## 🔹 Step 8: QuickSight Dashboard for OU-Level FinOps

### AWS QuickSight dashboards can be deployed via CloudFormation, boto3, or template JSON. Below I’ll give you the JSON template plus deployment steps.

## 8.1: QuickSight Dataset (Athena Data Source)

## First, create a dataset in QuickSight that joins your FOCUS data with OU mapping.

### SQL for dataset:

```sql 
SELECT f.year,
       f.month,
       f.usageAccountId,
       m.AccountName,
       m.OU,
       f.serviceCode,
       f.region,
       f.effectiveCost,
       f.listCost,
       f.billedCost,
       f.usageAmount,
       f.unit
FROM focus_org_billing f
LEFT JOIN account_ou_mapping m
  ON f.usageAccountId = m.AccountId
```



## 8.2: Dashboard Template (JSON)

### Save this JSON as focus-ou-dashboard.json.
```json 
{
  "Version": "1",
  "DashboardId": "Focus-OU-Dashboard",
  "Name": "FOCUS OU Dashboard",
  "SourceEntity": {
    "SourceTemplate": {
      "DataSetReferences": [
        {
          "DataSetPlaceholder": "FOCUS_OU",
          "DataSetArn": "arn:aws:quicksight:us-east-1:YOUR_ACCOUNT_ID:dataset/FOCUS_OU"
        }
      ],
      "Arn": "arn:aws:quicksight:us-east-1:YOUR_ACCOUNT_ID:template/FOCUS_OU_Template"
    }
  },
  "Permissions": [
    {
      "Principal": "arn:aws:quicksight:us-east-1:YOUR_ACCOUNT_ID:user/default/YOUR_USER",
      "Actions": [
        "quicksight:DescribeDashboard",
        "quicksight:ListDashboardVersions",
        "quicksight:QueryDashboard",
        "quicksight:UpdateDashboardPermissions",
        "quicksight:UpdateDashboard"
      ]
    }
  ]
}
```



## 8.3: Key Visuals in the Dashboard

## The dashboard will include:

### Spend by OU (bar chart)
```sql 
SELECT OU, SUM(effectiveCost) AS total_cost
FROM FOCUS_OU
GROUP BY OU
ORDER BY total_cost DESC;
```
### Spend Trend by OU (line chart)

```sql
SELECT year, month, OU, SUM(effectiveCost) AS total_cost
FROM FOCUS_OU
GROUP BY year, month, OU
ORDER BY year, month;
```



## Top Accounts in Each OU (heatmap)
```sql
SELECT OU, AccountName, SUM(effectiveCost) AS account_cost
FROM FOCUS_OU
GROUP BY OU, AccountName
ORDER BY account_cost DESC;
```



### Service Breakdown by OU (stacked bar chart)
```sql
SELECT OU, serviceCode, SUM(effectiveCost) AS service_cost
FROM FOCUS_OU
GROUP BY OU, serviceCode;
```

## Discount Efficiency (list vs effective cost)
```sql
SELECT OU,
       SUM(listCost) AS list_cost,
       SUM(effectiveCost) AS effective_cost,
       (1 - SUM(effectiveCost)/SUM(listCost))*100 AS discount_pct
FROM FOCUS_OU
GROUP BY OU;
```


## 8.4: Deploy the Dashboard
## AWS CLI:
```sql 
aws quicksight create-dashboard \
  --aws-account-id YOUR_ACCOUNT_ID \
  --cli-input-json file://focus-ou-dashboard.json
```



### boto3 (Python):
```python
import boto3, json

qs = boto3.client("quicksight")

with open("focus-ou-dashboard.json") as f:
    dashboard_def = json.load(f)

resp = qs.create_dashboard(
    AwsAccountId="YOUR_ACCOUNT_ID",
    DashboardId="Focus-OU-Dashboard",
    Name="FOCUS OU Dashboard",
    SourceEntity=dashboard_def["SourceEntity"],
    Permissions=dashboard_def["Permissions"]
)
print(resp)

```



### ✅ End Result

### Daily refreshed OU mappings → via Lambda → S3 → Athena.

### FOCUS billing data → centralized across all accounts.

### QuickSight OU Dashboard → visuals for:

### Spend by OU

### Spend trend over time

### Top accounts per OU

### Service breakdown per OU

### Discount efficiency



