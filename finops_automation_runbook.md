# 📘 FinOps Automation Runbook & Executive Summary

---

## 📊 Executive Summary (Non-Technical)

The FinOps automation we’ve built provides **daily financial visibility** into AWS spend, with a strong focus on **budgets, forecasts, and accountability per Organizational Unit (OU)**. It combines proactive alerts, daily reporting, and long-term historical tracking.

### What Leadership Gets:
- 📧 **Daily Executive Snapshot**: PDF of the FinOps dashboard in QuickSight.
- 🟩🟨🟥 **Traffic Light KPIs**: Each OU is marked Red, Yellow, or Green depending on budget health.
- 🔎 **Drill-Through Dashboards**: Click into an OU for service- and account-level details.
- 🔮 **Forecasting**: Machine learning predictions (3–6 months).
- 🛡 **Assurance**: All daily cost data is archived for audit and compliance.
- 🚨 **Alerts**: Leadership is notified by email if the automation ever fails.

👉 In short: Executives always have **current, accurate, and actionable cost insights** without manual effort.

---

## 🛠 Technical Deployment Runbook

### Step 1. Prerequisites
- AWS account with IAM admin rights
- QuickSight Enterprise Edition
- Athena + Glue enabled
- SES verified domain/email for digest delivery
- Terraform + AWS CLI installed

### Step 2. S3 Buckets
```hcl
resource "aws_s3_bucket" "finops_summaries" {
  bucket = "finops-daily-summaries"
}

resource "aws_s3_bucket_versioning" "finops_summaries_versioning" {
  bucket = aws_s3_bucket.finops_summaries.id
  versioning_configuration { status = "Enabled" }
}
```

### Step 3. IAM Role for Lambda
Includes access to S3, Athena, Glue, SES, QuickSight, and CloudWatch Logs.

### Step 4. Athena + Glue Setup
Athena external table partitioned by `year/month/day`, backed by Parquet files in S3.

```sql
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
```

### Step 5. Lambda (Daily Digest)
- Runs Athena query → generates CSV (OU breakdown)
- Archives CSV as Parquet (Snappy) in S3
- Registers Athena partition
- Triggers QuickSight dataset refresh
- Sends digest email with CSV attached

### Step 6. Scheduling
CloudWatch Event Rule: `cron(0 7 * * ? *)` → triggers Lambda daily.

### Step 7. QuickSight Setup
- Dataset: Athena → finops_db.ou_breakdown
- SPICE import for performance
- Dashboard visuals:
  - Combo chart (Actual vs Budget vs Forecast)
  - KPI traffic lights (Red/Yellow/Green per OU)
  - Drill-through: OU → Service → Account
- Subscriptions:
  - Execs: PDF snapshot
  - FinOps: CSV digest (from Lambda)

### Step 8. Monitoring & Alerts
- CloudWatch Alarm → SNS topic → Email to FinOps team
- SES bounce/complaint → same SNS

```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "finops-digest-lambda-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.finops_alerts.arn]
  dimensions = { FunctionName = aws_lambda_function.finops_digest.function_name }
}
```

### Step 9. Disaster Recovery (DR)
- If a daily run fails:
  1. Backfill with Athena `UNLOAD` into Parquet in S3.
  2. Re-register partition with `ALTER TABLE`.
  3. Trigger QuickSight ingestion with CLI.
- Weekly Athena check for missing partitions.

```sql
SELECT year, month, day, COUNT(*)
FROM finops_db.ou_breakdown
GROUP BY year, month, day
ORDER BY year, month, day;
```

---

## ✅ Final State

- 📧 FinOps team: **Daily CSV digest** + archived Parquet in S3.
- 📊 Executives: **Auto-refreshed QuickSight dashboard & PDF snapshots**.
- 🚨 Failures trigger **email alerts**.
- 🛡 Full DR playbook ensures **no gaps** in cost visibility.
- 🔮 Long-term forecasts + OU budget health at a glance.

👉 Outcome: A fully automated, auditable, and leadership-friendly **FinOps cost visibility platform**.

