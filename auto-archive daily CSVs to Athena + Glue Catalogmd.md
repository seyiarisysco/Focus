# auto-archive daily CSVs to Athena + Glue Catalog, so you build a history of FinOps reports that can be queried long-term (great for audits & trend analysis).

## Perfect 🚀 — let’s add historical archiving of daily FinOps summaries so you can query months or years of reports directly in Athena. This ensures you have a long-term audit trail beyond just QuickSight snapshots or emails.

## 🔹 1. Approach

Every daily summary CSV is already stored in S3 (s3://finops-daily-summaries/daily-summaries/).

We’ll create a Glue Crawler + Glue Catalog Table to make these CSVs queryable in Athena.

Partition data by BillingDate so queries run fast.

FinOps team can run queries like:

```sql
SELECT BillingDate, OU, Service, SUM(Cost)
FROM finops_daily_summary
WHERE BillingDate BETWEEN date '2025-01-01' AND date '2025-03-31'
GROUP BY BillingDate, OU, Service;
```



## 🔹 2. S3 Bucket Organization

Ensure the Lambda writes files like this:
```bash
s3://finops-daily-summaries/daily-summaries/BillingDate=2025-09-14/cost-summary.csv
s3://finops-daily-summaries/daily-summaries/BillingDate=2025-09-15/cost-summary.csv
```



That way, BillingDate becomes a partition key in Glue.

## 🔹 3. Glue Catalog + Crawler (Terraform)

```hcl
resource "aws_glue_catalog_database" "finops" {
  name = "finops_db"
}

resource "aws_glue_crawler" "daily_summary" {
  name         = "finops-daily-summary-crawler"
  role         = aws_iam_role.glue_service_role.arn
  database_name = aws_glue_catalog_database.finops.name

  s3_target {
    path = "s3://finops-daily-summaries/daily-summaries/"
  }

  configuration = jsonencode({
    Version = 1.0
    Grouping = {
      TableGroupingPolicy = "CombineCompatibleSchemas"
    }
  })

  schedule = "cron(0 7 * * ? *)" # Run every day at 7AM UTC
}

```


## 🔹 4. IAM Role for Glue
```hcl
resource "aws_iam_role" "glue_service_role" {
  name = "glue-finops-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = { Service = "glue.amazonaws.com" },
        Action    = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "glue_s3_access" {
  role       = aws_iam_role.glue_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role_policy_attachment" "glue_athena_access" {
  role       = aws_iam_role.glue_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonAthenaFullAccess"
}

```


## 🔹 5. Resulting Athena Table

After first crawler run, you’ll have a table like:

```sql
CREATE EXTERNAL TABLE finops_db.finops_daily_summary (
  OU string,
  Service string,
  Cost double
)
PARTITIONED BY (BillingDate date)
STORED AS PARQUET
LOCATION 's3://finops-daily-summaries/daily-summaries/';
```



## 🔹 6. Benefits

✅ Audit trail: Every daily CSV is preserved forever
✅ Trend analysis: Query spend across months/years
✅ Integration: Can feed into BI tools beyond QuickSight
✅ Performance: Partitioning by BillingDate keeps queries cheap & fast

✅ End Result

Your FinOps workflow now has:

🚨 Real-time anomaly alerts (SNS → email)

📧 Daily spend digest (with Top 10 + CSV)

📨 Daily PDF dashboard snapshots (QuickSight subscription)

📂 Historical archive in S3 + Athena/Glue (long-term reporting & audits)



