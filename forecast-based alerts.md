👉 Do you want me to also add forecast-based alerts (e.g., notify FinOps if projected spend will exceed budget by >10%)?

Perfect ✅ — let’s extend your FinOps automation to include forecast-based alerts.
This way, the system not only shows projections in QuickSight, but also notifies FinOps if future costs are forecasted to exceed budget thresholds.

🔹 1. Approach

Use QuickSight Forecasts for visualization.

Run a separate Athena job (via Lambda) to compute rolling monthly forecasts (same dataset).

Compare forecasted spend vs. budget thresholds (per OU or total).

Publish alerts via SNS (email, just like anomaly alerts).

🔹 2. Athena Query for Budget Forecasting

We’ll forecast using historical monthly cost, but since QuickSight’s forecast is visual-only, we’ll replicate a simple forecast in Athena with a moving average / linear trend:

WITH monthly AS (
  SELECT date_trunc('month', BillingDate) AS Month,
         OU,
         SUM(Cost) AS TotalCost
  FROM finops_db.finops_daily_summary
  GROUP BY 1, OU
)
SELECT OU,
       approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
       avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg,
       date_add('month', 1, max(Month)) AS NextMonth
FROM monthly;


This gives:

ForecastHigh = upper bound (95th percentile of history)

ForecastAvg = expected spend

NextMonth = projected billing period

🔹 3. Lambda for Forecast Alerts (forecast_alerts.py)

import boto3, os

athena = boto3.client("athena")
sns = boto3.client("sns")

def run_athena_query(query, database="finops_db"):
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": database},
        ResultConfiguration={"OutputLocation": os.environ["ATHENA_OUTPUT"]}
    )
    exec_id = response["QueryExecutionId"]

    while True:
        status = athena.get_query_execution(QueryExecutionId=exec_id)
        state = status["QueryExecution"]["Status"]["State"]
        if state in ["SUCCEEDED", "FAILED", "CANCELLED"]:
            break
    if state != "SUCCEEDED":
        raise Exception(f"Athena query failed: {state}")

    result = athena.get_query_results(QueryExecutionId=exec_id)
    return result

def handler(event, context):
    query = """
    WITH monthly AS (
      SELECT date_trunc('month', BillingDate) AS Month,
             OU,
             SUM(Cost) AS TotalCost
      FROM finops_daily_summary
      GROUP BY 1, OU
    )
    SELECT OU,
           approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
           avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg,
           date_add('month', 1, max(Month)) AS NextMonth
    FROM monthly;
    """

    result = run_athena_query(query)
    rows = result["ResultSet"]["Rows"][1:]  # skip header

    alerts = []
    budget = float(os.environ.get("BUDGET_THRESHOLD", "10000"))  # default $10k

    for row in rows:
        ou, forecast_high, forecast_avg, next_month = [c.get("VarCharValue") for c in row["Data"]]
        if float(forecast_high) > budget:
            alerts.append(f"⚠️ OU {ou} forecasted to exceed budget in {next_month}: est ${forecast_high}")

    if alerts:
        message = "\n".join(alerts)
        sns.publish(
            TopicArn=os.environ["SNS_TOPIC"],
            Subject="AWS Forecast Alert: Budget Risk",
            Message=message
        )

    return {"alerts_sent": len(alerts)}


🔹 4. Terraform — Hook It Up

resource "aws_lambda_function" "forecast_alerts" {
  function_name = "forecast-alerts"
  runtime       = "python3.12"
  handler       = "forecast_alerts.handler"
  role          = aws_iam_role.metric_lambda_role.arn

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      SNS_TOPIC     = aws_sns_topic.finops_alerts.arn
      ATHENA_OUTPUT = "s3://finops-daily-summaries/athena-results/"
      BUDGET_THRESHOLD = "10000" # default, can override per OU later
    }
  }
}

resource "aws_cloudwatch_event_rule" "monthly_forecast" {
  name                = "monthly-forecast-alert"
  schedule_expression = "cron(0 6 1 * ? *)" # run 1st of every month at 6AM UTC
}

resource "aws_cloudwatch_event_target" "forecast_lambda" {
  rule      = aws_cloudwatch_event_rule.monthly_forecast.name
  target_id = "ForecastLambda"
  arn       = aws_lambda_function.forecast_alerts.arn
}

resource "aws_lambda_permission" "allow_forecast_events" {
  statement_id  = "AllowExecutionFromEvents"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.forecast_alerts.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.monthly_forecast.arn
}

🔹 5. End Result

Now your FinOps alerting pipeline includes:

🚨 Anomaly Alerts (real-time, spikes >30%)

📧 Daily Digest (Top 10 OUs, Services, CSV link)

📨 Exec PDF Snapshot (QuickSight subscription)

📂 Historical archive + Athena views (daily, monthly, quarterly, yearly)

🔮 ML-powered Forecasts in QuickSight (visual)

⚠️ Forecast-based Alerts via Lambda (notify if future spend > budget)


👉 Do you want me to make the budget thresholds OU-specific (e.g., Marketing $15k, DevOps $20k, Finance $8k), or keep it as a single global budget cap?

Perfect ✅ — let’s make budget thresholds OU-specific, so each Organizational Unit (OU) is compared against its own budget cap instead of a global threshold.


🔹 1. Approach

Maintain a budget mapping table in Athena (or DynamoDB).

Join forecast results against budget per OU.

Alert only when a forecasted OU exceeds its own defined budget.

We’ll keep it in Athena (S3 + Glue) so everything stays queryable and FinOps can update budgets easily.

🔹 2. Budget Mapping Table in S3

Upload a CSV to S3 (s3://finops-daily-summaries/ou-budgets/ou_budgets.csv):

OU,Budget
Marketing,15000
DevOps,20000
Finance,8000
DataScience,12000
HR,5000

🔹 3. Glue Table for OU Budgets (Terraform)

resource "aws_glue_catalog_table" "ou_budgets" {
  name          = "ou_budgets"
  database_name = aws_glue_catalog_database.finops.name

  table_type = "EXTERNAL_TABLE"

  storage_descriptor {
    location      = "s3://finops-daily-summaries/ou-budgets/"
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
    serde_info {
      serialization_library = "org.apache.hadoop.hive.serde2.lazy.LazySerDe"
      parameters = {
        "serialization.format" = ","
        "field.delim"          = ","
      }
    }

    columns {
      name = "OU"
      type = "string"
    }
    columns {
      name = "Budget"
      type = "double"
    }
  }
}

🔹 4. Updated Forecast Query

Instead of a global threshold, join forecast with OU budgets:

WITH monthly AS (
  SELECT date_trunc('month', BillingDate) AS Month,
         OU,
         SUM(Cost) AS TotalCost
  FROM finops_daily_summary
  GROUP BY 1, OU
),
forecast AS (
  SELECT OU,
         approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
         avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg,
         date_add('month', 1, max(Month)) AS NextMonth
  FROM monthly
)
SELECT f.OU,
       f.NextMonth,
       f.ForecastHigh,
       b.Budget
FROM forecast f
JOIN ou_budgets b ON f.OU = b.OU
WHERE f.ForecastHigh > b.Budget;


🔹 5. Lambda for OU-Specific Forecast Alerts

import boto3, os

athena = boto3.client("athena")
sns = boto3.client("sns")

def run_athena_query(query, database="finops_db"):
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={"Database": database},
        ResultConfiguration={"OutputLocation": os.environ["ATHENA_OUTPUT"]}
    )
    exec_id = response["QueryExecutionId"]

    while True:
        status = athena.get_query_execution(QueryExecutionId=exec_id)
        state = status["QueryExecution"]["Status"]["State"]
        if state in ["SUCCEEDED", "FAILED", "CANCELLED"]:
            break
    if state != "SUCCEEDED":
        raise Exception(f"Athena query failed: {state}")

    result = athena.get_query_results(QueryExecutionId=exec_id)
    return result

def handler(event, context):
    query = """
    WITH monthly AS (
      SELECT date_trunc('month', BillingDate) AS Month,
             OU,
             SUM(Cost) AS TotalCost
      FROM finops_daily_summary
      GROUP BY 1, OU
    ),
    forecast AS (
      SELECT OU,
             approx_percentile(TotalCost, 0.95) OVER (PARTITION BY OU) AS ForecastHigh,
             avg(TotalCost) OVER (PARTITION BY OU) AS ForecastAvg,
             date_add('month', 1, max(Month)) AS NextMonth
      FROM monthly
    )
    SELECT f.OU, f.NextMonth, f.ForecastHigh, b.Budget
    FROM forecast f
    JOIN ou_budgets b ON f.OU = b.OU
    WHERE f.ForecastHigh > b.Budget;
    """

    result = run_athena_query(query)
    rows = result["ResultSet"]["Rows"][1:]  # skip header

    alerts = []
    for row in rows:
        ou, next_month, forecast_high, budget = [c.get("VarCharValue") for c in row["Data"]]
        alerts.append(
            f"⚠️ OU {ou} forecasted {next_month}: est ${float(forecast_high):,.2f} (Budget ${float(budget):,.2f})"
        )

    if alerts:
        sns.publish(
            TopicArn=os.environ["SNS_TOPIC"],
            Subject="AWS Forecast Budget Alert",
            Message="\n".join(alerts)
        )

    return {"alerts_sent": len(alerts)}

🔹 6. Example Alert Email

Subject: AWS Forecast Budget Alert

⚠️ OU Marketing forecasted 2025-10-01: est $15,400 (Budget $15,000)
⚠️ OU Finance forecasted 2025-10-01: est $9,200 (Budget $8,000)

✅ End Result

Now your forecast alerts are OU-specific 🎯

Each OU has its own budget threshold (stored in Athena via Glue)

Forecast query compares predicted spend vs OU budget

Alerts only fire for OUs at risk of overspending

FinOps can update budgets by simply uploading a new ou_budgets.csv

