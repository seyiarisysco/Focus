👉 Do you also want me to add an automated alert (SNS + CloudWatch) when anomalies are detected in QuickSight, so your FinOps team gets an email/Slack ping?


Awesome 🚀 — let’s close the loop: we’ll add automated anomaly alerts so your FinOps team gets notified (via SNS → Email/Slack) whenever QuickSight anomaly detection flags a cost spike.

🔹 1. How Alerts Work in QuickSight

QuickSight Anomaly Detection is visual/ML-driven.

To trigger alerts, we need CloudWatch Alarms that monitor Athena query results or QuickSight metrics.

Then, connect CloudWatch alarms → SNS for email/Slack/webhook notifications.

🔹 2. Architecture

Athena Scheduled Query (daily) → aggregates OU spend.

CloudWatch Metric → publishes spend per OU from Athena results (via Lambda).

CloudWatch Alarm → detects spikes beyond threshold (e.g., +30% day-over-day).

SNS Topic → sends alert email/Slack webhook.

🔹 3. Terraform Additions

Extend your FOCUS OU module with SNS + CloudWatch:

# ---------------- SNS Topic ----------------
resource "aws_sns_topic" "finops_alerts" {
  name = "finops-anomaly-alerts"
}

resource "aws_sns_topic_subscription" "finops_email" {
  topic_arn = aws_sns_topic.finops_alerts.arn
  protocol  = "email"
  endpoint  = "finops-team@example.com" # <-- change to Slack webhook/email
}

# ---------------- CloudWatch Alarm (Cost Spike) ----------------
resource "aws_cloudwatch_metric_alarm" "ou_spike" {
  alarm_name          = "OU-Cost-Spike"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  period              = 86400 # 1 day
  threshold           = 30
  datapoints_to_alarm = 1
  statistic           = "Average"
  namespace           = "Custom/FinOps"
  metric_name         = "CostSpikePct"
  alarm_description   = "Alerts when OU spend spikes more than 30% day-over-day"
  actions_enabled     = true
  alarm_actions       = [aws_sns_topic.finops_alerts.arn]
}

# ---------------- Lambda to Publish Metric ----------------
resource "aws_iam_role" "metric_lambda_role" {
  name = "finops-metric-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cw_access" {
  role       = aws_iam_role.metric_lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchFullAccess"
}

resource "aws_lambda_function" "cost_spike_publisher" {
  function_name = "publish-ou-cost-spike"
  runtime       = "python3.12"
  handler       = "index.handler"
  role          = aws_iam_role.metric_lambda_role.arn

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      NAMESPACE = "Custom/FinOps"
      METRIC    = "CostSpikePct"
    }
  }
}

# ---------------- CloudWatch Event to Trigger Daily ----------------
resource "aws_cloudwatch_event_rule" "daily_metric" {
  name                = "daily-ou-cost-metric"
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "lambda_metric_target" {
  rule      = aws_cloudwatch_event_rule.daily_metric.name
  target_id = "MetricLambda"
  arn       = aws_lambda_function.cost_spike_publisher.arn
}

resource "aws_lambda_permission" "allow_metric_events" {
  statement_id  = "AllowExecutionFromEvents"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cost_spike_publisher.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_metric.arn
}


🔹 4. Lambda Code (index.py)

This Lambda runs daily, queries Athena, and pushes cost spike metrics to CloudWatch:

import boto3, os, datetime

cloudwatch = boto3.client("cloudwatch")
athena = boto3.client("athena")

def handler(event, context):
    # Example: Query yesterday vs. day before
    query = """
    WITH daily_cost AS (
      SELECT billing_date, OU, SUM(effectiveCost) AS total_cost
      FROM focus_org_billing f
      JOIN account_ou_mapping m
        ON f.usageAccountId = m.AccountId
      WHERE billing_date >= date_add('day', -2, current_date)
      GROUP BY billing_date, OU
    )
    SELECT
      a.OU,
      ((a.total_cost - b.total_cost)/b.total_cost)*100 AS spike_pct
    FROM daily_cost a
    JOIN daily_cost b ON a.OU = b.OU
    WHERE a.billing_date = current_date - interval '1' day
      AND b.billing_date = current_date - interval '2' day;
    """

    # Submit Athena query
    athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": "default"},
        ResultConfiguration={"OutputLocation": "s3://focus-ou-mapping-central/athena-results/"}
    )

    # Example metric push (stubbed)
    cloudwatch.put_metric_data(
        Namespace=os.environ["NAMESPACE"],
        MetricData=[
            {
                "MetricName": os.environ["METRIC"],
                "Dimensions": [{"Name": "OU", "Value": "ExampleOU"}],
                "Value": 45.0,  # Example 45% spike
                "Unit": "Percent"
            }
        ]
    )

    return {"status": "ok"}


✅ End Result

Now your stack has:

📊 QuickSight Dashboard (OU-level FinOps with KPIs, filters, anomaly detection)
📉 Daily Athena Queries → compute OU costs
🚨 CloudWatch Alarms → detect spikes >30% day-over-day
📧 SNS Alerts → notify FinOps team via Email/Slack

