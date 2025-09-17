👉 Do you want me to now put everything together into a step-by-step deployment runbook (Terraform + Lambda + Athena + QuickSight) so your team can implement it end-to-end?

Perfect ✅ — let’s build a Step-by-Step Deployment Runbook for your end-to-end FinOps automation. This covers Terraform infra, Lambda jobs, Athena, QuickSight, SES, S3, and automation wiring.

🚀 FinOps Automation Deployment Runbook
🔹 Step 1. Prerequisites

AWS Account with IAM admin rights

QuickSight Enterprise Edition enabled

Athena + Glue available in region

SES Verified domain/email for sending digests

Terraform CLI + AWS CLI installed

🔹 Step 2. S3 Buckets (Terraform)

resource "aws_s3_bucket" "finops_summaries" {
  bucket = "finops-daily-summaries"
}

resource "aws_s3_bucket_versioning" "finops_summaries_versioning" {
  bucket = aws_s3_bucket.finops_summaries.id
  versioning_configuration {
    status = "Enabled"
  }
}


🔹 Step 3. IAM Roles for Lambda

resource "aws_iam_role" "metric_lambda_role" {
  name = "finops-metric-lambda-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "lambda_policy" {
  role = aws_iam_role.metric_lambda_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Action = ["s3:*"], Effect = "Allow", Resource = ["arn:aws:s3:::finops-daily-summaries", "arn:aws:s3:::finops-daily-summaries/*"] },
      { Action = ["athena:*"], Effect = "Allow", Resource = "*" },
      { Action = ["glue:*"], Effect = "Allow", Resource = "*" },
      { Action = ["ses:SendRawEmail"], Effect = "Allow", Resource = "*" },
      { Action = ["quicksight:CreateIngestion","quicksight:DescribeIngestion"], Effect = "Allow", Resource = "*" },
      { Action = ["logs:*"], Effect = "Allow", Resource = "*" }
    ]
  })
}


🔹 Step 4. Athena + Glue Setup

Glue Database

resource "aws_glue_catalog_database" "finops" {
  name = "finops_db"
}

Athena Table (Partitioned Parquet)

CREATE EXTERNAL TABLE finops_db.ou_breakdown (
  BillingDate date,
  OU string,
  Service string,
  LinkedAccount string,
  DailyCost double
)
PARTITIONED BY (year int, month int, day int)
STORED AS PARQUET
LOCATION 's3://finops-daily-summaries/ou-breakdowns/';


🔹 Step 5. Lambda for Daily Digest

Tasks:

Run Athena query → generate CSV for email

Archive data as Parquet in S3

Register new partition in Athena

Trigger QuickSight dataset refresh

Send email digest with CSV attached

(Simplified handler below)

def handler(event, context):
    # 1. Run Athena + download CSV
    exec_id = run_athena_query(QUERY)
    csv_file = download_results(exec_id)

    # 2. Convert CSV → Parquet + archive
    archive_parquet(csv_file)

    # 3. Register partition in Athena
    today = date.today()
    add_athena_partition(today.year, today.month, today.day)

    # 4. Refresh QuickSight dataset
    refresh_quicksight_dataset(os.environ["QUICKSIGHT_DATASET_ID"], os.environ["AWS_ACCOUNT_ID"])

    # 5. Send digest email with CSV attached
    send_digest_email(csv_file)

    return {"status": "Daily FinOps Digest Sent ✅"}


🔹 Step 6. Schedule Daily Run

resource "aws_cloudwatch_event_rule" "daily_digest" {
  name                = "daily-finops-digest"
  schedule_expression = "cron(0 7 * * ? *)" # 7AM UTC
}

resource "aws_cloudwatch_event_target" "digest_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_digest.name
  target_id = "DigestLambda"
  arn       = aws_lambda_function.finops_digest.arn
}


🔹 Step 7. QuickSight Setup

Dataset → Athena (finops_db.ou_breakdown).

SPICE Import → Faster dashboard refresh.

Dashboard visuals:

📊 Budget vs Forecast (Combo Chart + Conditional formatting)

🟩🟨🟥 KPI Traffic Lights per OU

🔎 Drill-through: OU → Service/Account table + line chart

Subscriptions:

Execs → Daily PDF snapshot

FinOps → Already covered by Lambda digest

🔹 Step 8. Validation

Run terraform apply

Manually trigger Lambda → check:

📧 Email received with CSV

📂 Parquet file in S3 with correct partition path

🔎 Athena query returns today’s data

📊 QuickSight dashboard shows refreshed costs

✅ End State

You now have a production-ready FinOps automation pipeline:

🚨 Anomaly detection + alerts

📧 Daily digest with OU CSV attachment

📂 S3 archive (Parquet, versioned, partitioned)

🗂 Athena partitions auto-registered

⚡ QuickSight dataset auto-refreshed

🟩🟨🟥 KPI traffic lights + drill-through dashboards
📨 Exec PDF snapshots always up-to-date

🛡 Disaster Recovery (DR) — FinOps Pipeline

🔹 1. Common Failure Scenarios

Lambda fails → Daily digest email not sent, Parquet not archived.

SES quota issues → Email suppressed/bounced.

Athena query timeout → No CSV generated.

Partition registration skipped → Data in S3 but not queryable.

QuickSight ingestion fails → Dashboard stale.

🔹 2. Reprocessing Historical Data

We keep S3 as the source of truth. If a daily run fails, you can reprocess missed days manually or via a backfill Lambda.

Athena Query to Backfill CSV/Parquet

Example: regenerate OU breakdown for Sept 10, 2025

UNLOAD (
    SELECT 
        date_trunc('day', BillingDate) AS BillingDate,
        OU,
        Service,
        LinkedAccount,
        SUM(Cost) AS DailyCost
    FROM finops_db.finops_daily_summary
    WHERE BillingDate = DATE '2025-09-10'
    GROUP BY 1, OU, Service, LinkedAccount
)
TO 's3://finops-daily-summaries/ou-breakdowns/year=2025/month=09/day=10/'
WITH (format = 'PARQUET', compression = 'SNAPPY');


This recreates the Parquet file in the correct partition path.

🔹 3. Register Missing Partition

If file exists but wasn’t registered:

ALTER TABLE finops_db.ou_breakdown
ADD IF NOT EXISTS PARTITION (year=2025, month=9, day=10)
LOCATION 's3://finops-daily-summaries/ou-breakdowns/year=2025/month=09/day=10/';

🔹 4. Resend Digest Email (Optional)

You can trigger the digest Lambda manually with a custom event payload:

{
  "rerun_date": "2025-09-10"
}

{
  "rerun_date": "2025-09-10"
}


Extend Lambda handler:

rerun_date = event.get("rerun_date")
if rerun_date:
    # Backfill specific date
    query = f"""
    SELECT date_trunc('day', BillingDate) AS BillingDate,
           OU, Service, LinkedAccount,
           SUM(Cost) AS DailyCost
    FROM finops_daily_summary
    WHERE BillingDate = DATE '{rerun_date}'
    GROUP BY 1, OU, Service, LinkedAccount
    """
    exec_id = run_athena_query(query)
    ...


🔹 5. QuickSight Refresh

If ingestion fails:

aws quicksight create-ingestion \
  --aws-account-id <ACCOUNT_ID> \
  --data-set-id <DATASET_ID> \
  --ingestion-id backfill-20250910


This forces QuickSight to reload fresh data.

🔹 6. Recommended DR Strategy

Daily: Monitor Lambda logs & SES delivery reports.

Weekly: Run Athena query to check for gaps:

SELECT year, month, day, COUNT(*) 
FROM finops_db.ou_breakdown
GROUP BY year, month, day
ORDER BY year, month, day;

Missing partitions = missed days → reprocess.

Monthly: Archive CSV digests in S3 Glacier Deep Archive (for compliance).

✅ DR Playbook Summary

📂 S3 = source of truth (daily costs always recoverable).

🔄 Backfill with Athena UNLOAD into partition path.

🗂 ALTER TABLE to re-register partitions.

📧 Manual rerun of digest Lambda to resend missed emails.

📊 Force QuickSight ingestion if dashboard is stale.

👉 Do you want me to also add a CloudWatch alarm + Slack/Email alert if the daily Lambda fails (so FinOps is notified same-day instead of discovering gaps later)?

Excellent ✅ — let’s add CloudWatch alarms + Slack/Email alerts so your FinOps team knows immediately if the daily pipeline fails.

🔹 1. What We’ll Monitor

Lambda Execution Failures (daily digest job)

Lambda Duration/Timeouts (e.g., Athena query too long)

SES Send Failures (email delivery issues)

🔹 2. CloudWatch Alarm for Lambda Failures

Terraform example:

resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "finops-digest-lambda-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alarm if FinOps digest Lambda fails"
  dimensions = {
    FunctionName = aws_lambda_function.finops_digest.function_name
  }
  alarm_actions = [aws_sns_topic.finops_alerts.arn]
}

🔹 3. SNS Topic for Alerts

resource "aws_sns_topic" "finops_alerts" {
  name = "finops-alerts"
}

resource "aws_sns_topic_subscription" "finops_email" {
  topic_arn = aws_sns_topic.finops_alerts.arn
  protocol  = "email"
  endpoint  = "finops-team@example.com"
}

🔹 4. Slack Integration (Optional)

If you want alerts in Slack:

Use AWS Chatbot or an SNS → Lambda → Slack webhook.

Example Slack Lambda snippet:

import json, urllib3, os

http = urllib3.PoolManager()
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK"]

def handler(event, context):
    message = event["Records"][0]["Sns"]["Message"]
    payload = {"text": f":rotating_light: FinOps Lambda Alert :rotating_light:\n{message}"}
    http.request("POST", SLACK_WEBHOOK, body=json.dumps(payload), headers={"Content-Type": "application/json"})


Attach SNS → Slack Lambda subscription.

🔹 5. SES Delivery Failures

Enable SNS notifications on SES bounces/complaints:

resource "aws_ses_identity_notification_topic" "ses_bounces" {
  identity  = "finops-reports@example.com"
  notification_type = "Bounce"
  topic_arn = aws_sns_topic.finops_alerts.arn
}

Now if SES can’t deliver a digest, FinOps will be alerted.

🔹 6. End Result

🚨 Lambda failure = Slack & Email alert within 5 minutes

🚨 SES delivery failure = Slack & Email alert

✅ FinOps team knows same-day → can trigger DR (rerun/backfill).

✅ Final FinOps Automation (with DR + Alerts)

📧 Daily digest (CSV + Parquet archive + QuickSight refresh)

📂 S3 versioned archive (Parquet, partitioned)

🗂 Athena partitions auto-registered

⚡ QuickSight dataset auto-refreshed

🟩🟨🟥 KPI dashboards + drill-through

🚨 CloudWatch alarms → Slack/Email alerts on failures

🛡 DR playbook: backfill missed days + re-ingest

ChatGPT said:

Perfect 👍 — we’ll keep the alerts email-only for now (simple and reliable).

✅ Finalized FinOps Automation Runbook
📊 Data Flow

Lambda (daily @7 AM UTC)

Runs Athena query → generates OU-level CSV

Archives as Parquet (Snappy) in S3 (partitioned, versioned)

Registers new Athena partition

Triggers QuickSight SPICE refresh

Sends daily digest email with CSV attached

Executives

Receive QuickSight PDF snapshot (auto-refreshed with SPICE)

FinOps Team

Gets daily digest email with CSV attachment

Uses S3 historical archive + Athena for deep dives

🚨 Monitoring & Alerts

CloudWatch Alarm → Monitors Errors metric for Lambda

SNS Topic (finops-alerts) → Sends email to FinOps distribution list

SES Bounces/Complaints → Also forward to same SNS topic

Terraform snippet (already included):

resource "aws_sns_topic" "finops_alerts" {
  name = "finops-alerts"
}

resource "aws_sns_topic_subscription" "finops_email" {
  topic_arn = aws_sns_topic.finops_alerts.arn
  protocol  = "email"
  endpoint  = "finops-team@example.com"
}

resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "finops-digest-lambda-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alert if FinOps digest Lambda fails"
  dimensions = {
    FunctionName = aws_lambda_function.finops_digest.function_name
  }
  alarm_actions = [aws_sns_topic.finops_alerts.arn]
}


🛡 Disaster Recovery (DR)

If daily job fails:

Run Athena UNLOAD query → regenerate CSV/Parquet for missed day

Re-register partition:

ALTER TABLE finops_db.ou_breakdown
ADD IF NOT EXISTS PARTITION (year=2025, month=9, day=15)
LOCATION 's3://finops-daily-summaries/ou-breakdowns/year=2025/month=09/day=15/';


Trigger QuickSight ingestion:

aws quicksight create-ingestion \
  --aws-account-id <ACCOUNT_ID> \
  --data-set-id <DATASET_ID> \
  --ingestion-id backfill-20250915

If email not delivered → SES will forward bounce/complaint alerts to FinOps team email via SNS.

🎯 Final Outcome

📧 Daily digest email → CSV (all OUs)

📂 Parquet archive → Versioned, partitioned, queryable in Athena

🗂 Partitions auto-registered

⚡ QuickSight auto-refreshed

🟩🟨🟥 KPI dashboards + drill-through

🚨 Email alerts on failures

🛡 DR playbook for reprocessing

