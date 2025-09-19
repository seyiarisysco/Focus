# 👉 Do you also want me to push this CSV automatically into QuickSight as a dataset so the daily summary is visible inside the dashboard too?

## Great move 🎯 — let’s push the daily summary CSV into QuickSight so your FinOps team sees it inside the dashboard as well as in email.

## 🔹 1. Approach

We already generate a daily CSV of spend by OU + Service.

Store it in S3 (finops-daily-summaries).

Register that S3 location as a QuickSight dataset (SPICE refresh daily).

Create a “Daily Summary” sheet in your existing FOCUS OU Dashboard.

## 🔹 2. Terraform — QuickSight S3 Data Source

```hcl
# S3 bucket already exists (finops-daily-summaries)

resource "aws_quicksight_data_source" "daily_summary_s3" {
  data_source_id = "daily-summary-s3"
  name           = "DailySummaryS3"
  aws_account_id = var.account_id
  type           = "S3"

  data_source_parameters {
    s3 {
      manifest_file_location {
        bucket = aws_s3_bucket.summary_reports.bucket
        key    = "manifest.json"
      }
    }
  }
}

```


## 🔹 3. Manifest File for QuickSight

Save this as manifest.json in your S3 bucket:
```json
{
  "fileLocations": [
    {
      "URIPrefixes": [
        "s3://finops-daily-summaries/daily-summaries/"
      ]
    }
  ],
  "globalUploadSettings": {
    "format": "CSV",
    "delimiter": ",",
    "textqualifier": "\"",
    "containsHeader": "true"
  }
}
```



## 🔹 4. QuickSight Dataset for Daily Summary
```hcl
resource "aws_quicksight_data_set" "daily_summary" {
  data_set_id    = "daily-summary-dataset"
  name           = "Daily Summary"
  aws_account_id = var.account_id
  import_mode    = "SPICE"

  physical_table_map = {
    "daily_summary_s3" = {
      s3_source = {
        data_source_arn = aws_quicksight_data_source.daily_summary_s3.arn
        input_columns = [
          { name = "BillingDate", type = "DATETIME" },
          { name = "OU", type = "STRING" },
          { name = "Service", type = "STRING" },
          { name = "Cost", type = "DECIMAL" }
        ]
      }
    }
  }
}
```



## 🔹 5. Add “Daily Summary” Sheet to Dashboard

Extend your focus-ou-template.json with a new sheet:
```json

{
  "Sheets": [
    {
      "Name": "Daily Summary",
      "Visuals": [
        {
          "TableVisual": {
            "Title": { "Text": "Top 10 OUs - Daily Spend" },
            "ChartConfiguration": {
              "FieldWells": {
                "TableAggregatedFieldWells": {
                  "GroupBy": [
                    { "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "OU" } }
                  ],
                  "Values": [
                    {
                      "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "Cost" },
                      "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                    }
                  ]
                }
              }
            }
          }
        },
        {
          "TableVisual": {
            "Title": { "Text": "Top 10 Services - Daily Spend" },
            "ChartConfiguration": {
              "FieldWells": {
                "TableAggregatedFieldWells": {
                  "GroupBy": [
                    { "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "Service" } }
                  ],
                  "Values": [
                    {
                      "Column": { "DataSetIdentifier": "Daily Summary", "ColumnName": "Cost" },
                      "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                    }
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
```



## 🔹 6. Glue Together in Terraform

Update dashboard definition so both datasets are referenced:
```yaml
resource "aws_quicksight_dashboard" "focus_ou" {
  dashboard_id   = "focus-ou-dashboard"
  name           = "FOCUS OU Dashboard"
  aws_account_id = var.account_id

  source_entity {
    source_template {
      arn = aws_quicksight_template.focus_ou.arn
      data_set_references = [
        {
          data_set_arn         = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        },
        {
          data_set_arn         = aws_quicksight_data_set.daily_summary.arn
          data_set_placeholder = "Daily Summary"
        }
      ]
    }
  }
}
```



## ✅ End Result

Now your FinOps workflow delivers value in two channels:

📧 Daily Email:

Summary KPIs

Top 10 OUs + Services

Link to CSV

📊 QuickSight Dashboard:

Existing OU insights + anomaly detection

New “Daily Summary” sheet with Top 10 tables

Data refreshes daily from S3

# 👉 Do you want me to also add a QuickSight email subscription (dashboard snapshots emailed daily to execs), so leadership gets visuals in their inbox without opening QuickSight?

## Perfect ✅ — let’s give your leadership team daily dashboard snapshots by email. This uses QuickSight Subscriptions, which automatically email PDF/PNG reports on a schedule.


## 🔹 1. How It Works

QuickSight dashboards → subscriptions

Send PDF snapshot daily/weekly to a distribution list

Uses QuickSight Enterprise Edition + email integration

Complements your FinOps daily summary emails

## 🔹 2. Terraform for QuickSight Dashboard Subscription

Terraform doesn’t yet have native QuickSight subscription resources, but we can use the AWS CLI / boto3 via a null_resource + local-exec (or a Lambda).

Here’s a Terraform example using aws cli:
```hcl
resource "null_resource" "quicksight_subscription" {
  provisioner "local-exec" {
    command = <<EOT
    aws quicksight create-dashboard-subscription \
      --aws-account-id ${var.account_id} \
      --dashboard-id focus-ou-dashboard \
      --name "FOCUS OU Daily Subscription" \
      --schedule "DAILY" \
      --recipients finops-team@example.com execs@example.com \
      --format PDF
    EOT
  }
}
```



⚠️ Replace recipients with a list of corporate emails (users must be in QuickSight).


## 🔹 3. Alternative — Python boto3 Script (More Flexible)
```python
import boto3

qs = boto3.client("quicksight")

response = qs.create_dashboard_subscription(
    AwsAccountId="123456789012",
    DashboardId="focus-ou-dashboard",
    Name="FOCUS OU Daily",
    Schedule="DAILY",
    Recipients=["finops-team@example.com", "execs@example.com"],
    Format="PDF"
)

print("Subscription created:", response)
```



This can be run once, or wrapped into a Lambda triggered by Terraform.

## 🔹 4. Recommended Setup

Recipients:

finops-team@example.com → FinOps engineers

execs@example.com → Leadership distribution list

Frequency: Daily at 9 AM (local time)

Format: PDF (cleaner for executives)

## ✅ End Result

Now you have a triple delivery model:

🚨 Anomaly Alerts → Email when cost spikes >30%

📧 Daily Spend Digest → Email summary with CSV + Top 10 OUs/Services

📨 Dashboard Subscription → PDF snapshot of QuickSight dashboard (for leadership)

This means:

FinOps team = gets actionable alerts + raw data

Executives = get visual summaries in their inbox

Dashboard = remains the single source of truth for deep dives

✅ What’s Done in Your Stack

 FOCUS OU Dashboard with filters, KPIs, anomaly detection

 Email anomaly alerts (>30% spike) via CloudWatch + SNS

 Daily summary email (KPIs + CSV link + Top 10 OU/Services)


 Daily QuickSight PDF subscription for executives
