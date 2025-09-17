👉 Do you want me to wire this directly INTO emails?

Perfect 👍 — then we’ll keep the anomaly alerting simple and reliable via email.

Here’s how it fits into your stack now:

🔹 1. What We Already Added

Athena Query → computes OU daily spend

Lambda Publisher → pushes % spike metric into CloudWatch

CloudWatch Alarm → triggers when cost spike >30%

SNS Topic → sends alerts

🔹 2. Email Subscription (Terraform)

We’ll finalize the SNS setup so your FinOps team receives email alerts:

resource "aws_sns_topic" "finops_alerts" {
  name = "finops-anomaly-alerts"
}

resource "aws_sns_topic_subscription" "finops_email" {
  topic_arn = aws_sns_topic.finops_alerts.arn
  protocol  = "email"
  endpoint  = "finops-team@example.com" # <-- replace with your FinOps distribution list
}


🔹 3. Deployment Flow

Terraform Apply → creates SNS topic + email subscription.

AWS will send a confirmation email to finops-team@example.com.

⚠️ You must click Confirm subscription to activate alerts.

Once confirmed, every CloudWatch Alarm trigger will send an email like:

ALARM: "OU-Cost-Spike" in US-EAST-1
Threshold: >30% increase in OU spend
Metric: Custom/FinOps - CostSpikePct
Triggered for OU: <OU_Name>


🔹 4. Optional Fine-Tuning

Thresholds per OU → If some OUs normally fluctuate more, we can set different alarm thresholds per OU.

Multiple emails → Add more aws_sns_topic_subscription blocks with different endpoints.

HTML formatting → SNS supports rich messages if you prefer more structured alerts.

✅ End Result

Exec Dashboard in QuickSight (KPIs, OU filters, date range slider, anomaly insights)

Daily anomaly detection via Athena + CloudWatch

Automatic email alerts when OU spend spikes unexpectedly

👉 Do you want me to also add a daily cost summary email (not just anomaly alerts) — e.g., “Yesterday’s total spend: $X, Top OU: Y, % change vs. day before”?

Perfect 🚀 — let’s add a daily cost summary email alongside anomaly alerts.
This way, your FinOps team gets a digest every morning with:

Yesterday’s total spend

% change vs. day before

Top OU by cost

Top Service by cost

🔹 1. Approach

We’ll extend your Lambda publisher (which already queries Athena for anomalies) so it also:

Runs a daily cost summary Athena query

Formats results into an email-friendly message

Publishes it to the same SNS Topic (finops-anomaly-alerts)

🔹 2. Updated Lambda Code (index.py)

import boto3, os, datetime

athena = boto3.client("athena")
sns = boto3.client("sns")
OUTPUT = "s3://focus-ou-mapping-central/athena-results/"  # change to your bucket

def run_athena_query(query, database="default"):
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": database},
        ResultConfiguration={"OutputLocation": OUTPUT}
    )
    execution_id = response["QueryExecutionId"]

    # Wait for completion
    while True:
        status = athena.get_query_execution(QueryExecutionId=execution_id)
        state = status["QueryExecution"]["Status"]["State"]
        if state in ["SUCCEEDED", "FAILED", "CANCELLED"]:
            break
    if state != "SUCCEEDED":
        raise Exception(f"Athena query failed: {state}")

    result = athena.get_query_results(QueryExecutionId=execution_id)
    return result

def handler(event, context):
    # Query: Yesterday vs. Day Before
    query = """
    WITH daily AS (
      SELECT billing_date, OU, serviceCode, SUM(effectiveCost) AS cost
      FROM focus_org_billing f
      JOIN account_ou_mapping m ON f.usageAccountId = m.AccountId
      WHERE billing_date >= date_add('day', -2, current_date)
      GROUP BY billing_date, OU, serviceCode
    )
    SELECT billing_date, OU, serviceCode, SUM(cost) AS total_cost
    FROM daily
    GROUP BY billing_date, OU, serviceCode
    ORDER BY billing_date, total_cost DESC
    """

    result = run_athena_query(query)

    # Parse results
    rows = result["ResultSet"]["Rows"]
    header = [col["VarCharValue"] for col in rows[0]["Data"]]
    data = [[col.get("VarCharValue", "0") for col in row["Data"]] for row in rows[1:]]

    # Summarize yesterday
    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)

    yesterday_data = [r for r in data if r[0] == str(yesterday)]
    day_before = today - datetime.timedelta(days=2)
    day_before_data = [r for r in data if r[0] == str(day_before)]

    total_yesterday = sum(float(r[3]) for r in yesterday_data)
    total_day_before = sum(float(r[3]) for r in day_before_data) or 1
    pct_change = ((total_yesterday - total_day_before) / total_day_before) * 100

    # Top OU
    top_ou = max(yesterday_data, key=lambda r: float(r[3]))[1]
    # Top Service
    top_service = max(yesterday_data, key=lambda r: float(r[3]))[2]

    # Email body
    message = f"""
    📊 Daily AWS Spend Summary ({yesterday})

    Total Spend: ${total_yesterday:,.2f}
    Change vs. Day Before: {pct_change:+.2f}%

    Top OU: {top_ou}
    Top Service: {top_service}

    — FOCUS OU FinOps Bot
    """

    sns.publish(
        TopicArn=os.environ["SNS_TOPIC"],
        Subject=f"AWS Daily Cost Summary - {yesterday}",
        Message=message
    )

    return {"status": "sent", "total": total_yesterday}


🔹 3. Terraform for Lambda + SNS

Update your Lambda definition:

resource "aws_lambda_function" "cost_summary" {
  function_name = "daily-cost-summary"
  runtime       = "python3.12"
  handler       = "index.handler"
  role          = aws_iam_role.metric_lambda_role.arn

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      SNS_TOPIC = aws_sns_topic.finops_alerts.arn
    }
  }
}

# Daily trigger
resource "aws_cloudwatch_event_rule" "daily_summary" {
  name                = "daily-cost-summary"
  schedule_expression = "cron(0 8 * * ? *)" # 8 AM UTC daily
}

resource "aws_cloudwatch_event_target" "summary_target" {
  rule      = aws_cloudwatch_event_rule.daily_summary.name
  target_id = "CostSummaryLambda"
  arn       = aws_lambda_function.cost_summary.arn
}

resource "aws_lambda_permission" "allow_summary_events" {
  statement_id  = "AllowDailySummaryEvents"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cost_summary.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_summary.arn
}

✅ End Result

Now your FinOps alerts system does two things:

🚨 Anomaly Alerts

Trigger when OU spend spikes >30%

Email subject: ALARM: OU-Cost-Spike

📊 Daily Cost Summary Email (8 AM UTC)
Example email:

📊 Daily AWS Spend Summary (2025-09-14)

Total Spend: $12,345.67
Change vs. Day Before: +18.5%

Top OU: Marketing-OU
Top Service: AmazonEC2

— FOCUS OU FinOps Bot


👉 Do you also want me to enhance this daily email with an attached CSV or table of Top 10 OUs & Services for deeper visibility (not just the #1)?


Perfect ✅ — let’s enhance the daily FinOps email so your team gets not only the high-level KPIs but also a Top 10 OU and Top 10 Service breakdown in CSV + inline table format.

🔹 1. Enhanced Email Content

The daily email will now include:

Total Spend and % change vs. day before

Top OU and Top Service

Inline Table → Top 10 OUs by spend

Inline Table → Top 10 Services by spend

CSV Attachment with full daily breakdown

🔹 2. Updated Lambda Code (index.py)

import boto3, os, datetime, csv, io

athena = boto3.client("athena")
sns = boto3.client("sns")
OUTPUT = "s3://focus-ou-mapping-central/athena-results/"  # change to your bucket

def run_athena_query(query, database="default"):
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": database},
        ResultConfiguration={"OutputLocation": OUTPUT}
    )
    execution_id = response["QueryExecutionId"]

    # Wait until complete
    while True:
        status = athena.get_query_execution(QueryExecutionId=execution_id)
        state = status["QueryExecution"]["Status"]["State"]
        if state in ["SUCCEEDED", "FAILED", "CANCELLED"]:
            break
    if state != "SUCCEEDED":
        raise Exception(f"Athena query failed: {state}")

    result = athena.get_query_results(QueryExecutionId=execution_id)
    return result

def handler(event, context):
    # Query yesterday’s spend by OU and Service
    query = """
    SELECT billing_date, OU, serviceCode, SUM(effectiveCost) AS total_cost
    FROM focus_org_billing f
    JOIN account_ou_mapping m ON f.usageAccountId = m.AccountId
    WHERE billing_date >= date_add('day', -2, current_date)
    GROUP BY billing_date, OU, serviceCode
    ORDER BY billing_date, total_cost DESC
    """
    result = run_athena_query(query)

    # Parse Athena results
    rows = result["ResultSet"]["Rows"]
    data = [[col.get("VarCharValue", "0") for col in row["Data"]] for row in rows[1:]]  # skip header

    today = datetime.date.today()
    yesterday = str(today - datetime.timedelta(days=1))
    day_before = str(today - datetime.timedelta(days=2))

    # Filter for yesterday
    y_data = [r for r in data if r[0] == yesterday]
    d_data = [r for r in data if r[0] == day_before]

    # Totals
    total_yesterday = sum(float(r[3]) for r in y_data)
    total_day_before = sum(float(r[3]) for r in d_data) or 1
    pct_change = ((total_yesterday - total_day_before) / total_day_before) * 100

    # Top 10 OUs
    ou_totals = {}
    for _, ou, _, cost in y_data:
        ou_totals[ou] = ou_totals.get(ou, 0) + float(cost)
    top_ous = sorted(ou_totals.items(), key=lambda x: x[1], reverse=True)[:10]

    # Top 10 Services
    svc_totals = {}
    for _, _, svc, cost in y_data:
        svc_totals[svc] = svc_totals.get(svc, 0) + float(cost)
    top_svcs = sorted(svc_totals.items(), key=lambda x: x[1], reverse=True)[:10]

    # Inline Table (text)
    ou_table = "\n".join([f"{i+1}. {ou}: ${amt:,.2f}" for i, (ou, amt) in enumerate(top_ous)])
    svc_table = "\n".join([f"{i+1}. {svc}: ${amt:,.2f}" for i, (svc, amt) in enumerate(top_svcs)])

    # Build CSV
    csv_buf = io.StringIO()
    writer = csv.writer(csv_buf)
    writer.writerow(["BillingDate", "OU", "Service", "Cost"])
    for row in y_data:
        writer.writerow(row)
    csv_body = csv_buf.getvalue()

    # Message
    message = f"""
📊 Daily AWS Spend Summary ({yesterday})

Total Spend: ${total_yesterday:,.2f}
Change vs. Day Before: {pct_change:+.2f}%

Top OUs:
{ou_table}

Top Services:
{svc_table}

— FOCUS OU FinOps Bot
"""

    # Send SNS with email + CSV attachment (SNS doesn’t support attachments natively)
    # Option 1: Inline CSV link (recommended)
    s3 = boto3.client("s3")
    bucket = os.environ.get("SNS_CSV_BUCKET", "focus-ou-mapping-central")
    key = f"daily-summaries/cost-summary-{yesterday}.csv"
    s3.put_object(Bucket=bucket, Key=key, Body=csv_body)

    presigned_url = s3.generate_presigned_url(
        "get_object", Params={"Bucket": bucket, "Key": key}, ExpiresIn=86400
    )

    message += f"\n📂 Full CSV: {presigned_url}"

    sns.publish(
        TopicArn=os.environ["SNS_TOPIC"],
        Subject=f"AWS Daily Cost Summary - {yesterday}",
        Message=message
    )

    return {"status": "sent", "total": total_yesterday}


🔹 3. Terraform Additions

Add an S3 bucket for CSV storage (if not reusing your OU bucket):

resource "aws_s3_bucket" "summary_reports" {
  bucket = "finops-daily-summaries"
}


Update the Lambda environment:

environment {
  variables = {
    SNS_TOPIC     = aws_sns_topic.finops_alerts.arn
    SNS_CSV_BUCKET = aws_s3_bucket.summary_reports.bucket
  }
}

✅ End Result

Your FinOps team will now receive a daily email like this:

EXAMPLE EMAIL:

Subject:
AWS Daily Cost Summary - 2025-09-14

Body:

📊 Daily AWS Spend Summary (2025-09-14)

Total Spend: $12,345.67
Change vs. Day Before: +18.5%

Top OUs:
1. Marketing-OU: $4,500.12
2. DevOps-OU: $3,200.55
...

Top Services:
1. AmazonEC2: $6,789.33
2. AmazonS3: $2,450.99
...

📂 Full CSV: https://s3.amazonaws.com/finops-daily-summaries/cost-summary-2025-09-14.csv?X-Amz-Expires=86400

⚡ Now you have:

Anomaly alerts (spikes >30%) via email

Daily spend summary email with Top OUs + Services

Downloadable CSV for deeper analysis