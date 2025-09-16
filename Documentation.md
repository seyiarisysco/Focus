# FinOps AWS Implementation Documentation

## Table of Contents
1. [Executive Overview](#executive-overview)
2. [Engineering View](#engineering-view)
3. [Automation Code Examples](#automation-code-examples)
   - [Terraform for Budgets](#terraform-for-budgets)
   - [Python (boto3) for CUR](#python-boto3-for-cur)
   - [AWS CLI for RI Purchase](#aws-cli-for-ri-purchase)
   - [SQL for Athena Analysis](#sql-for-athena-analysis)
4. [Appendix](#appendix)

---

## Executive Overview

This section provides a **high-level view** of AWS FinOps implementation.  
See also: [Engineering View](#engineering-view).

![Executive View](FinOps_AWS_Executive_View.png)

**Key Components**
- AWS Account and linked OUs
- Cost and Usage Reports (CUR)
- Budgets
- Dashboards for executives

---

## Engineering View

This section details the **technical workflows**.  
See also: [Executive Overview](#executive-overview).

![Engineering View](FinOps_AWS_Engineering_View.png)

**Key Elements**
- Numbered swimlanes (1–n) for process stages
- Bold connectors labeled with data formats (CSV, Parquet, Alert Email)
- Centralized legend explaining symbols and flows

---

## Automation Code Examples

### Terraform for Budgets
```terraform
resource "aws_budgets_budget" "monthly_cost" {
  name              = "Monthly-Cost-Budget"
  budget_type       = "COST"
  limit_amount      = "1000"
  limit_unit        = "USD"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator = "GREATER_THAN"
    threshold           = 80
    threshold_type      = "PERCENTAGE"
    notification_type   = "FORECASTED"
    subscriber_email_addresses = ["finops-team@example.com"]
  }
}
```

### Python (boto3) for CUR
```python
import boto3

client = boto3.client("cur")

response = client.put_report_definition(
    ReportDefinition={
        "ReportName": "daily-cur",
        "TimeUnit": "DAILY",
        "Format": "textORcsv",
        "Compression": "GZIP",
        "S3Bucket": "my-cur-bucket",
        "S3Prefix": "reports/",
        "S3Region": "us-east-1",
        "AdditionalSchemaElements": ["RESOURCES"]
    }
)
print("CUR report created:", response)
```

### AWS CLI for RI Purchase
```bash
aws ec2 purchase-reserved-instances-offering     --reserved-instances-offering-id <offering-id>     --instance-count 1
```

### SQL for Athena Analysis
```sql
SELECT
    line_item_product_code,
    SUM(line_item_unblended_cost) AS total_cost
FROM
    cur_database.cur_table
WHERE
    year = '2025' AND month = '09'
GROUP BY
    line_item_product_code
ORDER BY
    total_cost DESC;
```

---

## Appendix

- Editable diagrams: `FinOps_AWS_Views.drawio`  
- Exported images: `FinOps_AWS_Executive_View.png`, `FinOps_AWS_Engineering_View.png`  
- Use [diagrams.net](https://app.diagrams.net) for editing  
