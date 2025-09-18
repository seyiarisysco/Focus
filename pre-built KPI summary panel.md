# 👉 Do you want me to also add a pre-built KPI summary panel (e.g., Total Cost, % Savings from Discounts, Top OU by Spend) so you have an executive-level summary on top of the dashboard?

# Perfect 🎯 — let’s finish this by adding a KPI summary panel so your QuickSight OU Dashboard has an executive-level overview before the detailed charts.

## 🔹 1. KPI Metrics to Show

### We’ll add three high-level KPIs at the top of the dashboard:

### Total Cost (SUM of effectiveCost)

### % Savings (Discount Efficiency) → (1 - SUM(effectiveCost)/SUM(listCost)) * 100

### Top OU by Spend (max of OU grouped spend)

## 🔹 2. JSON Snippet (KPI Visuals)

Add these to your focus-ou-template.json in the Visuals section of the first sheet:

{
  "KPIVisual": {
    "Title": { "Text": "Total Cost" },
    "ChartConfiguration": {
      "FieldWells": {
        "KPIAggregatedFieldWells": {
          "Values": [
            {
              "FieldId": "effectiveCost",
              "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" },
              "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
            }
          ]
        }
      }
    }
  }
},
{
  "KPIVisual": {
    "Title": { "Text": "Discount Efficiency (%)" },
    "ChartConfiguration": {
      "FieldWells": {
        "KPIAggregatedFieldWells": {
          "Values": [
            {
              "FieldId": "discountEfficiency",
              "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" },
              "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
            }
          ]
        }
      },
      "ConditionalFormatting": {
        "KPIConditionalFormattingOptions": [
          {
            "PrimaryValue": {
              "TextColor": { "Solid": { "Color": "#1f77b4" } }
            }
          }
        ]
      }
    }
  }
},
{
  "KPIVisual": {
    "Title": { "Text": "Top OU by Spend" },
    "ChartConfiguration": {
      "FieldWells": {
        "KPIAggregatedFieldWells": {
          "Values": [
            {
              "FieldId": "maxOUCost",
              "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" },
              "AggregationFunction": { "SimpleNumericalAggregation": "MAX" }
            }
          ],
          "GroupBy": [
            {
              "FieldId": "OU",
              "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" }
            }
          ]
        }
      }
    }
  }
}


🔹 3. Terraform — No Change in Structure

Your Terraform just needs to point to the updated template JSON.
Example:

resource "aws_quicksight_template" "focus_ou" {
  template_id    = "focus-ou-template"
  name           = "FOCUS OU Template"
  aws_account_id = var.account_id
  source_entity {
    source_analysis {
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:analysis/focus-ou-analysis"
      data_set_references = [
        {
          data_set_arn         = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }
}


✅ End Result

Your QuickSight FOCUS OU Dashboard will now look like this:

📊 KPI Summary (executive panel at top):

Total Cost → current spend (filtered by OU, Account, Service, Date Range)

Discount Efficiency (%) → % savings from list vs. effective cost

Top OU by Spend → which OU is driving the most cost

📉 Interactive Filters:

OU, Account, Service, Date Range

📈 Detailed Visuals:

Spend by OU

Spend trend by OU

Service breakdown by OU

Discount efficiency by OU


👉 Do you also want me to add an "anomaly detection widget" (QuickSight Insight visual) that automatically flags unusual cost spikes per OU?

🔥 Excellent — let’s supercharge your FOCUS OU Dashboard by adding an Anomaly Detection Widget (QuickSight Insight Visual) so it automatically flags unusual cost spikes per OU.

🔹 1. What’s an Insight Visual in QuickSight?

Insight Visuals use ML-based anomaly detection built into QuickSight.

They detect outliers or sudden changes in metrics like effectiveCost.

Perfect for FinOps anomaly monitoring.

We’ll configure one to:

Track spend trends per OU

Highlight sudden spikes/drops in cost

🔹 2. JSON Snippet for Anomaly Detection

Add this to your focus-ou-template.json under Visuals:

{
  "InsightVisual": {
    "Title": { "Text": "Anomaly Detection: Cost by OU" },
    "ChartConfiguration": {
      "InsightConfiguration": {
        "ComputationConfigurations": [
          {
            "ComputationId": "AnomalyDetection",
            "AnomalyDetection": {
              "TimeSeries": {
                "TimeField": {
                  "FieldId": "billing_date",
                  "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "billing_date" }
                },
                "ValueField": {
                  "FieldId": "effectiveCost",
                  "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }
                }
              },
              "Category": {
                "FieldId": "OU",
                "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" }
              }
            }
          }
        ]
      }
    }
  }
}

🔹 3. Supporting Athena Dataset

We already added billing_date (synthetic date from year + month):

CAST(DATE_PARSE(CONCAT(year, '-', month, '-01'), '%Y-%m-%d') AS DATE) AS billing_date


This makes it time-series ready for anomaly detection.

🔹 4. Terraform Integration

No changes in Terraform structure — just update the template JSON:

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
        }
      ]
    }
  }
}


✅ End Result

Your FOCUS OU QuickSight Dashboard now includes:

📊 KPI Summary:

Total Cost

Discount Efficiency (%)

Top OU by Spend

📉 Interactive Filters:

OU, Account, Service, Date Range

📈 Detailed Visuals:

Spend by OU

Spend trend by OU

Service breakdown

Discount efficiency

🚨 Anomaly Detection Widget (Insight Visual):

Flags unexpected spikes/drops in OU spend

Helps FinOps teams catch cost anomalies before they balloon


👉 Do you also want me to add an automated alert (SNS + CloudWatch) when anomalies are detected in QuickSight, so your FinOps team gets an email/Slack ping?


