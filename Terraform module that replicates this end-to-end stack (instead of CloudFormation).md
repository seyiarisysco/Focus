👉 Would you like me to also generate a Terraform module that replicates this end-to-end stack (instead of CloudFormation)?

Got it ✅ — let’s build a Terraform module that replicates the end-to-end FOCUS + OU FinOps stack we just built in CloudFormation.

This module will deploy:

S3 buckets for FOCUS exports + OU mapping

Lambda + EventBridge to refresh OU mapping daily

Athena tables for FOCUS + OU mapping

Outputs you can plug into QuickSight

🔹 Terraform Module: modules/focus-ou

variable "focus_export_bucket" {
  default = "focus-data-export-central"
}

variable "ou_mapping_bucket" {
  default = "focus-ou-mapping-central"
}

variable "region" {
  default = "us-east-1"
}

provider "aws" {
  region = var.region
}

# ---------------- S3 Buckets ----------------
resource "aws_s3_bucket" "focus_export" {
  bucket = var.focus_export_bucket
  versioning { enabled = true }
}

resource "aws_s3_bucket" "ou_mapping" {
  bucket = var.ou_mapping_bucket
}

# ---------------- IAM Role ----------------
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

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "orgs_access" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSOrganizationsReadOnlyAccess"
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

# ---------------- Lambda ----------------
data "archive_file" "lambda_zip" {
  type        = "zip"
  source {
    content  = <<EOF
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
EOF
    filename = "index.py"
  }
  output_path = "lambda.zip"
}

resource "aws_lambda_function" "ou_mapping" {
  function_name = "focus-ou-mapping"
  runtime       = "python3.12"
  handler       = "index.handler"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 120

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      BUCKET = aws_s3_bucket.ou_mapping.bucket
    }
  }
}

# ---------------- EventBridge Schedule ----------------
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

# ---------------- Athena Tables ----------------
resource "aws_athena_named_query" "focus_billing" {
  name      = "CreateFocusBillingTable"
  database  = "default"
  workgroup = "primary"
  query     = <<EOT
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
LOCATION 's3://${var.focus_export_bucket}/focus/';
EOT
}

resource "aws_athena_named_query" "ou_mapping" {
  name      = "CreateOuMappingTable"
  database  = "default"
  workgroup = "primary"
  query     = <<EOT
CREATE EXTERNAL TABLE IF NOT EXISTS account_ou_mapping (
  AccountId string,
  AccountName string,
  OU string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://${var.ou_mapping_bucket}/';
EOT
}

# ---------------- Outputs ----------------
output "focus_export_bucket" {
  value = aws_s3_bucket.focus_export.bucket
}

output "ou_mapping_bucket" {
  value = aws_s3_bucket.ou_mapping.bucket
}

output "ou_mapping_lambda" {
  value = aws_lambda_function.ou_mapping.arn
}


🔹 Usage in Root main.tf


module "focus_ou" {
  source             = "./modules/focus-ou"
  focus_export_bucket = "my-org-focus-billing"
  ou_mapping_bucket   = "my-org-ou-mapping"
  region              = "us-east-1"
}


module "focus_ou" {
  source             = "./modules/focus-ou"
  focus_export_bucket = "my-org-focus-billing"
  ou_mapping_bucket   = "my-org-ou-mapping"
  region              = "us-east-1"
}


Run:

terraform init
terraform apply


✅ End Result

S3: Buckets for FOCUS exports + OU mappings

Lambda: Generates OU mapping CSV daily

Athena: Tables for FOCUS billing + OU mapping

QuickSight: You can connect to the Athena dataset and import the OU dashboard JSON


👉 Would you like me to also extend this module to auto-deploy the QuickSight OU dashboard (via aws_quicksight_dashboard resource), so it’s truly one-click from Terraform?


Perfect 🚀 — let’s make your Terraform module fully one-click by adding QuickSight OU Dashboard deployment on top of the FOCUS + OU pipeline.

⚠️ Note: The aws_quicksight_dashboard resource requires:

QuickSight Enterprise Edition

Your QuickSight user/namespace

An Athena data source + dataset already linked

We’ll extend the module to create:

Athena data source

Athena dataset (FOCUS + OU join)

QuickSight dashboard with OU visuals


🔹 Extended Terraform Module (modules/focus-ou)

variable "focus_export_bucket" { default = "focus-data-export-central" }
variable "ou_mapping_bucket"  { default = "focus-ou-mapping-central" }
variable "region"             { default = "us-east-1" }
variable "account_id"         {}
variable "quicksight_user"    {} # e.g. user/default/yourname

provider "aws" { region = var.region }

# ---------------- Athena Data Source ----------------
resource "aws_quicksight_data_source" "athena" {
  data_source_id = "focus-ou-athena"
  name           = "FOCUS_OU_Athena"
  type           = "ATHENA"

  aws_account_id = var.account_id

  parameters {
    athena {
      work_group = "primary"
    }
  }

  permissions {
    principal = "arn:aws:quicksight:${var.region}:${var.account_id}:${var.quicksight_user}"
    actions   = ["quicksight:DescribeDataSource", "quicksight:PassDataSource"]
  }
}

# ---------------- Athena Dataset ----------------
resource "aws_quicksight_data_set" "focus_ou" {
  data_set_id   = "focus-ou-dataset"
  name          = "FOCUS_OU"
  aws_account_id = var.account_id

  import_mode = "DIRECT_QUERY"

  physical_table_map = {
    "FOCUS_OU" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "FOCUS_OU_SQL"
        sql_query       = <<EOT
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
FROM default.focus_org_billing f
LEFT JOIN default.account_ou_mapping m
  ON f.usageAccountId = m.AccountId
EOT
        columns = [
          { name = "year",           type = "INTEGER" },
          { name = "month",          type = "INTEGER" },
          { name = "usageAccountId", type = "STRING" },
          { name = "AccountName",    type = "STRING" },
          { name = "OU",             type = "STRING" },
          { name = "serviceCode",    type = "STRING" },
          { name = "region",         type = "STRING" },
          { name = "effectiveCost",  type = "DECIMAL" },
          { name = "listCost",       type = "DECIMAL" },
          { name = "billedCost",     type = "DECIMAL" },
          { name = "usageAmount",    type = "DECIMAL" },
          { name = "unit",           type = "STRING" }
        ]
      }
    }
  }
}

# ---------------- QuickSight Dashboard ----------------
resource "aws_quicksight_dashboard" "focus_ou" {
  dashboard_id   = "focus-ou-dashboard"
  name           = "FOCUS OU Dashboard"
  aws_account_id = var.account_id

  source_entity {
    source_template {
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:template/focus-ou-template"
    }
  }

  permissions {
    principal = "arn:aws:quicksight:${var.region}:${var.account_id}:${var.quicksight_user}"
    actions   = ["quicksight:DescribeDashboard", "quicksight:QueryDashboard"]
  }
}



🔹 Root main.tf


module "focus_ou" {
  source            = "./modules/focus-ou"
  focus_export_bucket = "my-org-focus-billing"
  ou_mapping_bucket   = "my-org-ou-mapping"
  region              = "us-east-1"
  account_id          = "123456789012"
  quicksight_user     = "user/default/yourname"
}


🔹 Deploy

terraform init
terraform apply


✅ End Result

S3 → buckets for billing & OU mapping

Lambda → daily OU mapping refresh

Athena → tables for FOCUS + OU mapping

QuickSight → OU dashboard auto-deployed (line charts, bar charts, spend breakdowns)

👉 Do you want me to also create a sample QuickSight template definition JSON (with the charts: Spend by OU, Spend Trend, Service Breakdown, Discount %) that this Terraform can import automatically instead of a blank dashboard?


Perfect 🔥 — let’s add a QuickSight Template JSON with pre-built FOCUS OU visuals so that your Terraform stack can deploy a ready-to-use dashboard (not blank).


🔹 1. QuickSight OU Dashboard Template JSON

Save this as focus-ou-template.json.
This template defines 4 visuals:

Spend by OU (bar chart)

Spend Trend by OU (line chart)

Service Breakdown by OU (stacked bar chart)

Discount Efficiency (list vs effective cost)

{
  "AwsAccountId": "123456789012",
  "TemplateId": "focus-ou-template",
  "Name": "FOCUS OU Template",
  "SourceEntity": {
    "SourceAnalysis": {
      "Arn": "arn:aws:quicksight:us-east-1:123456789012:analysis/focus-ou-analysis",
      "DataSetReferences": [
        {
          "DataSetArn": "arn:aws:quicksight:us-east-1:123456789012:dataset/focus-ou-dataset",
          "DataSetPlaceholder": "FOCUS_OU"
        }
      ]
    }
  },
  "Permissions": [
    {
      "Principal": "arn:aws:quicksight:us-east-1:123456789012:user/default/YOUR_USER",
      "Actions": [
        "quicksight:DescribeTemplate",
        "quicksight:UpdateTemplate",
        "quicksight:DeleteTemplate",
        "quicksight:DescribeTemplatePermissions"
      ]
    }
  ],
  "Version": {
    "Description": "OU Dashboard v1",
    "Sheets": [
      {
        "Name": "OU Overview",
        "Visuals": [
          {
            "BarChartVisual": {
              "Title": { "Text": "Spend by OU" },
              "ChartConfiguration": {
                "FieldWells": {
                  "BarChartAggregatedFieldWells": {
                    "Category": [{ "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }],
                    "Values": [{ "FieldId": "effectiveCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } }]
                  }
                }
              }
            }
          },
          {
            "LineChartVisual": {
              "Title": { "Text": "Spend Trend by OU" },
              "ChartConfiguration": {
                "FieldWells": {
                  "LineChartAggregatedFieldWells": {
                    "Category": [{ "FieldId": "month", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "month" } }],
                    "Colors": [{ "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }],
                    "Values": [{ "FieldId": "effectiveCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } }]
                  }
                }
              }
            }
          },
          {
            "BarChartVisual": {
              "Title": { "Text": "Service Breakdown by OU" },
              "ChartConfiguration": {
                "FieldWells": {
                  "BarChartAggregatedFieldWells": {
                    "Category": [{ "FieldId": "serviceCode", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "serviceCode" } }],
                    "Colors": [{ "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }],
                    "Values": [{ "FieldId": "effectiveCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } }]
                  }
                }
              }
            }
          },
          {
            "TableVisual": {
              "Title": { "Text": "Discount Efficiency by OU" },
              "ChartConfiguration": {
                "FieldWells": {
                  "TableAggregatedFieldWells": {
                    "GroupBy": [{ "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }],
                    "Values": [
                      { "FieldId": "listCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "listCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } },
                      { "FieldId": "effectiveCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } }
                    ]
                  }
                }
              }
            }
          }
        ]
      }
    ]
  }
}


👉 Replace:

123456789012 with your AWS account ID

YOUR_USER with your QuickSight user name

🔹 2. Terraform to Import Template

Add this to your module:

resource "aws_quicksight_template" "focus_ou" {
  template_id    = "focus-ou-template"
  name           = "FOCUS OU Template"
  aws_account_id = var.account_id

  source_entity {
    source_analysis {
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:analysis/focus-ou-analysis"
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }

  permissions {
    principal = "arn:aws:quicksight:${var.region}:${var.account_id}:${var.quicksight_user}"
    actions   = ["quicksight:DescribeTemplate", "quicksight:UpdateTemplate", "quicksight:DeleteTemplate"]
  }
}

resource "aws_quicksight_dashboard" "focus_ou" {
  dashboard_id   = "focus-ou-dashboard"
  name           = "FOCUS OU Dashboard"
  aws_account_id = var.account_id

  source_entity {
    source_template {
      arn = aws_quicksight_template.focus_ou.arn
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }

  permissions {
    principal = "arn:aws:quicksight:${var.region}:${var.account_id}:${var.quicksight_user}"
    actions   = ["quicksight:DescribeDashboard", "quicksight:QueryDashboard"]
  }
}


🔹 3. Deploy


terraform init
terraform apply


After apply:

OU mapping Lambda + S3 = daily refresh

Athena dataset = cost + OU join

QuickSight dashboard = OU dashboard with spend, trends, services, discount

✅ End-to-end FinOps pipeline in one Terraform apply.
