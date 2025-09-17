👉 Do you want me to also automatically register/update the Athena partitions daily (instead of relying on the crawler), so queries are instantly up-to-date after each archive?

Perfect ✅ — let’s automate Athena partition updates so that as soon as the daily Parquet file is archived, the partition is registered. This way:

🚀 Queries run instantly, without waiting for the Glue crawler.

📂 Each day’s partition (year/month/day) is added automatically.

📝 Old partitions remain queryable (audit trail intact).

🔹 1. Lambda Partition Registration

After uploading the Parquet file, run an Athena DDL query to add the partition:

def add_athena_partition(year, month, day):
    query = f"""
    ALTER TABLE finops_db.ou_breakdown
    ADD IF NOT EXISTS PARTITION (year={year}, month={month}, day={day})
    LOCATION 's3://{os.environ["ARCHIVE_BUCKET"]}/ou-breakdowns/year={year}/month={month:02d}/day={day:02d}/'
    """
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": "finops_db"},
        ResultConfiguration={"OutputLocation": os.environ["ATHENA_OUTPUT"]}
    )
    return response["QueryExecutionId"]


Inside handler(), after archive_parquet(local_file):

today = date.today()
add_athena_partition(today.year, today.month, today.day)


🔹 2. IAM Permissions Needed

Update the Lambda role with Athena DDL permissions:

resource "aws_iam_role_policy" "lambda_athena" {
  role = aws_iam_role.metric_lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "athena:StartQueryExecution",
          "athena:GetQueryExecution"
        ],
        Effect   = "Allow",
        Resource = "*"
      },
      {
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ],
        Effect   = "Allow",
        Resource = [
          "arn:aws:s3:::finops-daily-summaries",
          "arn:aws:s3:::finops-daily-summaries/*"
        ]
      }
    ]
  })
}


🔹 3. Benefits

✅ Immediate availability: Daily partition is ready right after archive.

✅ No crawler lag: Crawler becomes optional (can still run weekly for schema validation).

✅ Efficient queries: Athena only scans relevant partitions.

🔹 4. Example Query (No Delay)

Right after daily run:

SELECT Service, SUM(DailyCost) AS Cost
FROM finops_db.ou_breakdown
WHERE OU = 'Marketing'
  AND year = 2025
  AND month = 09
  AND day = 15
GROUP BY Service;

🔹 5. End State (Locked In)

Your FinOps pipeline now provides:

📧 Daily digest email with OU CSV attached

📂 Versioned archive in S3 (Parquet + Snappy)

🗂 Daily partition registration in Athena (instant query availability)

🔍 Historical OU → Service → Account drill-through

✅ Full audit trail, optimized costs, fast BI queries

