##Perfect 🚀 — let’s make it practical. Below you’ll find CloudFormation + Terraform + Athena SQL examples to implement FOCUS in AWS.
CloudFormation Template (Deploy AWS Data Exports + S3 Bucket)

AWSTemplateFormatVersion: '2010-09-09'
Description: "Enable FOCUS Data Export to S3"

Resources:
  FocusExportBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "focus-data-export-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled

  FocusDataExport:
    Type: AWS::BillingConductor::BillingGroup
    Properties:
      Name: "FocusExportGroup"
      ComputationPreference:
        PricingPlanArn: !Ref PricingPlan
      PrimaryAccountId: !Ref AWS::AccountId
	  
👉 This is a starting point. In practice, you’ll need to set up an AWS Data Export resource (using AWS::DataExports::Export) once CloudFormation supports it (currently via API/Console).

2. Terraform Script (Export Billing Data with FOCUS Schema)

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "focus_export" {
  bucket = "focus-data-export-example"
}

resource "aws_s3_bucket_policy" "focus_export_policy" {
  bucket = aws_s3_bucket.focus_export.id
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
      "Resource": "${aws_s3_bucket.focus_export.arn}/*"
    }
  ]
}
EOT
}

resource "aws_cur_report_definition" "focus_export" {
  report_name                = "focus-export"
  time_unit                  = "DAILY"
  format                     = "Parquet"
  compression                = "Parquet"
  s3_bucket                  = aws_s3_bucket.focus_export.bucket
  s3_prefix                  = "focus"
  s3_region                  = "us-east-1"
  additional_schema_elements = ["RESOURCES"]
  report_versioning          = "CREATE_NEW_REPORT"
}



👉 This will export AWS CUR data. To align with FOCUS 1.0, you enable FOCUS schema in the Billing Console → Data Exports. Terraform support is rolling out — until then you can bootstrap with CUR.

3. Athena SQL Example (FOCUS Standardized Queries) Create Athena table for FOCUS export (Parquet in S3)

CREATE EXTERNAL TABLE IF NOT EXISTS focus_billing (
  lineItemId string,
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
  resourceId string
)
STORED AS PARQUET
LOCATION 's3://focus-data-export-example/focus/';


Example Queries Monthly Spend by Service

SELECT serviceCode,
       SUM(effectiveCost) as total_cost
FROM focus_billing
WHERE year = 2025 AND month = 8
GROUP BY serviceCode
ORDER BY total_cost DESC;

Discount Analysis (List vs Effective)

SELECT serviceCode,
       SUM(listCost) AS list_cost,
       SUM(effectiveCost) AS effective_cost,
       (1 - SUM(effectiveCost)/SUM(listCost)) * 100 AS discount_pct
FROM focus_billing
WHERE year = 2025 AND month = 8
GROUP BY serviceCode;


Top 10 Accounts by Spend

SELECT usageAccountId,
       SUM(effectiveCost) AS account_cost
FROM focus_billing
GROUP BY usageAccountId
ORDER BY account_cost DESC
LIMIT 10;

4. QuickSight Dashboard Deployment (FOCUS Dashboard)

AWS has pre-built Cloud Intelligence Dashboards (CID) for FOCUS

Deploy via AWS CLI:
aws cloudformation create-stack \
  --stack-name focus-dashboard \
  --template-url https://wellarchitectedlabs.s3.amazonaws.com/Cost/200_FinOps/Deployment/FOCUS/FOCUS-Dashboard.yaml \
  --capabilities CAPABILITY_NAMED_IAM

After deployment → open Amazon QuickSight, connect to Athena → focus_billing table.

Now you have:

CloudFormation/Terraform to set up S3 + exports,

Athena SQL queries mapped to FOCUS schema,

QuickSight dashboard for visualization.


