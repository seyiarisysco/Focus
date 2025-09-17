👉 Do you want me to also wire this OU drill-through CSV export into your daily FinOps digest email (so FinOps automatically gets the OU-level breakdown attached instead of manual export)?

Excellent ✅ — let’s extend the daily FinOps digest email to automatically include the OU-level CSV breakdown so FinOps engineers don’t need to manually export from QuickSight.

🔹 1. Approach

The OU drill-through table (Service + Account breakdown) already exists.

We’ll generate its CSV automatically via Athena query in Lambda (same dataset powering QuickSight).

Lambda will:

Run Athena query for all OUs (or top N OUs by cost).

Save results to S3.

Attach CSV(s) to the daily FinOps digest email via SES or presigned S3 link.

🔹 2. Athena Query for OU Drill-Through CSV

SELECT 
    date_trunc('day', BillingDate) AS BillingDate,
    OU,
    Service,
    LinkedAccount,
    SUM(Cost) AS DailyCost
FROM finops_db.finops_daily_summary
GROUP BY 1, OU, Service, LinkedAccount
ORDER BY BillingDate DESC, OU, DailyCost DESC;


🔹 3. Lambda Function (Python)

We extend your daily digest Lambda to generate the CSV:

import boto3, os, csv, tempfile
from datetime import date

athena = boto3.client("athena")
s3 = boto3.client("s3")
ses = boto3.client("ses")

def run_athena_query(query, database="finops_db"):
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": database},
        ResultConfiguration={"OutputLocation": os.environ["ATHENA_OUTPUT"]}
    )
    exec_id = response["QueryExecutionId"]
    status = "RUNNING"
    while status in ["RUNNING", "QUEUED"]:
        res = athena.get_query_execution(QueryExecutionId=exec_id)
        status = res["QueryExecution"]["Status"]["State"]
    if status != "SUCCEEDED":
        raise Exception("Athena query failed")
    return exec_id

def handler(event, context):
    query = """
    SELECT 
        date_trunc('day', BillingDate) AS BillingDate,
        OU,
        Service,
        LinkedAccount,
        SUM(Cost) AS DailyCost
    FROM finops_daily_summary
    GROUP BY 1, OU, Service, LinkedAccount
    ORDER BY BillingDate DESC, OU, DailyCost DESC
    """
    exec_id = run_athena_query(query)

    # Download results
    s3_path = f"{os.environ['ATHENA_OUTPUT']}{exec_id}.csv"
    local_file = f"/tmp/ou_breakdown_{date.today()}.csv"
    bucket = s3_path.split('/')[2]
    key = '/'.join(s3_path.split('/')[3:])
    s3.download_file(bucket, key, local_file)

    # Send email with attachment
    with open(local_file, "rb") as f:
        raw_data = f.read()

    ses.send_raw_email(
        Source=os.environ["SES_FROM"],
        Destinations=[os.environ["SES_TO"]],
        RawMessage={
            "Data": f"""From: {os.environ["SES_FROM"]}
To: {os.environ["SES_TO"]}
Subject: FinOps Daily Digest with OU Breakdown
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="NextPart"

--NextPart
Content-Type: text/plain

Attached is the OU-level cost breakdown (daily, service, account).
--NextPart
Content-Type: text/csv
Content-Disposition: attachment; filename="ou_breakdown_{date.today()}.csv"

{raw_data.decode('utf-8')}
--NextPart--
"""
        }
    )

    return {"status": "Email sent with OU breakdown CSV"}


🔹 4. Terraform Hook

resource "aws_lambda_function" "finops_digest" {
  function_name = "finops-daily-digest"
  runtime       = "python3.12"
  handler       = "digest.handler"
  role          = aws_iam_role.metric_lambda_role.arn

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      ATHENA_OUTPUT = "s3://finops-daily-summaries/athena-results/"
      SES_FROM      = "finops-reports@example.com"
      SES_TO        = "finops-team@example.com"
    }
  }
}

resource "aws_cloudwatch_event_rule" "daily_digest" {
  name                = "daily-finops-digest"
  schedule_expression = "cron(0 7 * * ? *)"
}

resource "aws_cloudwatch_event_target" "digest_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_digest.name
  target_id = "DigestLambda"
  arn       = aws_lambda_function.finops_digest.arn
}


🔹 5. End Result

Every morning:

📊 FinOps Daily Digest email now includes OU breakdown CSV attached.

The CSV matches the drill-through view in QuickSight.

Engineers can filter or pivot locally in Excel/Sheets without needing to log into QuickSight.

✅ Your FinOps Delivery Model Now

🚨 Real-time anomaly alerts

📧 Daily digest email

KPIs

Top 10 OUs & Services

Attached OU drill-through CSV ✅

📨 Exec PDF snapshot (QuickSight subscription)

📂 Historical archive (S3 + Athena + Glue)

📊 Trends + Roll-ups (Daily → Yearly)

🔮 ML-powered forecasts

⚠️ OU-specific budget alerts

🟩🟨🟥 Traffic light KPIs

🔎 Drill-through dashboards
🔟 📂 Auto CSV export in digest emails ✅

