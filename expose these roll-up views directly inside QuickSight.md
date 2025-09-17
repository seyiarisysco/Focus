Do you want me to also expose these roll-up views directly inside QuickSight (new sheets for Monthly, Quarterly, Yearly), so execs never need to touch Athena?

Perfect ✅ — let’s expose the Athena roll-up views directly in QuickSight, so execs see monthly, quarterly, yearly spend trends inside the dashboard without touching Athena.

🔹 1. Roll-up Views Recap (Athena)

We already created:

finops_db.monthly_summary

finops_db.quarterly_summary

finops_db.yearly_summary

Each provides aggregated costs by OU + Service.

🔹 2. QuickSight Datasets for Roll-up Views

In Terraform:

# Monthly Summary Dataset
resource "aws_quicksight_data_set" "monthly_summary" {
  data_set_id   = "monthly-summary-dataset"
  name          = "Monthly Summary"
  aws_account_id = var.account_id
  import_mode   = "DIRECT_QUERY"

  physical_table_map = {
    "monthly_summary" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "Monthly Summary SQL"
        sql_query       = "SELECT * FROM finops_db.monthly_summary"
      }
    }
  }
}

# Quarterly Summary Dataset
resource "aws_quicksight_data_set" "quarterly_summary" {
  data_set_id   = "quarterly-summary-dataset"
  name          = "Quarterly Summary"
  aws_account_id = var.account_id
  import_mode   = "DIRECT_QUERY"

  physical_table_map = {
    "quarterly_summary" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "Quarterly Summary SQL"
        sql_query       = "SELECT * FROM finops_db.quarterly_summary"
      }
    }
  }
}

# Yearly Summary Dataset
resource "aws_quicksight_data_set" "yearly_summary" {
  data_set_id   = "yearly-summary-dataset"
  name          = "Yearly Summary"
  aws_account_id = var.account_id
  import_mode   = "DIRECT_QUERY"

  physical_table_map = {
    "yearly_summary" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "Yearly Summary SQL"
        sql_query       = "SELECT * FROM finops_db.yearly_summary"
      }
    }
  }
}


🔹 3. Extend Dashboard Template (JSON)

Add new sheets to focus-ou-template.json:

{
  "Sheets": [
    {
      "Name": "Monthly Trends",
      "Visuals": [
        {
          "LineChartVisual": {
            "Title": { "Text": "Monthly Spend by OU" },
            "ChartConfiguration": {
              "FieldWells": {
                "LineChartAggregatedFieldWells": {
                  "Category": [
                    { "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "Month" } }
                  ],
                  "Values": [
                    {
                      "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "TotalCost" },
                      "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                    }
                  ],
                  "Colors": [
                    { "Column": { "DataSetIdentifier": "Monthly Summary", "ColumnName": "OU" } }
                  ]
                }
              }
            }
          }
        }
      ]
    },
    {
      "Name": "Quarterly Trends",
      "Visuals": [
        {
          "BarChartVisual": {
            "Title": { "Text": "Quarterly Spend by Service" },
            "ChartConfiguration": {
              "FieldWells": {
                "BarChartAggregatedFieldWells": {
                  "Category": [
                    { "Column": { "DataSetIdentifier": "Quarterly Summary", "ColumnName": "Quarter" } }
                  ],
                  "Values": [
                    {
                      "Column": { "DataSetIdentifier": "Quarterly Summary", "ColumnName": "TotalCost" },
                      "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                    }
                  ],
                  "Colors": [
                    { "Column": { "DataSetIdentifier": "Quarterly Summary", "ColumnName": "Service" } }
                  ]
                }
              }
            }
          }
        }
      ]
    },
    {
      "Name": "Yearly Trends",
      "Visuals": [
        {
          "PieChartVisual": {
            "Title": { "Text": "Yearly Spend by OU" },
            "ChartConfiguration": {
              "FieldWells": {
                "PieChartAggregatedFieldWells": {
                  "Category": [
                    { "Column": { "DataSetIdentifier": "Yearly Summary", "ColumnName": "OU" } }
                  ],
                  "Values": [
                    {
                      "Column": { "DataSetIdentifier": "Yearly Summary", "ColumnName": "TotalCost" },
                      "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                    }
                  ]
                }
              }
            }
          }
        }
      ]
    }
  ]
}


🔹 4. Update Dashboard References

Update Terraform so the dashboard knows about new datasets:

resource "aws_quicksight_dashboard" "focus_ou" {
  dashboard_id   = "focus-ou-dashboard"
  name           = "FOCUS OU Dashboard"
  aws_account_id = var.account_id

  source_entity {
    source_template {
      arn = aws_quicksight_template.focus_ou.arn
      data_set_references = [
        { data_set_arn = aws_quicksight_data_set.focus_ou.arn, data_set_placeholder = "FOCUS_OU" },
        { data_set_arn = aws_quicksight_data_set.daily_summary.arn, data_set_placeholder = "Daily Summary" },
        { data_set_arn = aws_quicksight_data_set.monthly_summary.arn, data_set_placeholder = "Monthly Summary" },
        { data_set_arn = aws_quicksight_data_set.quarterly_summary.arn, data_set_placeholder = "Quarterly Summary" },
        { data_set_arn = aws_quicksight_data_set.yearly_summary.arn, data_set_placeholder = "Yearly Summary" }
      ]
    }
  }
}


✅ End Result

Your FOCUS OU QuickSight Dashboard now has extra sheets for roll-ups:

📈 Monthly Trends → Line chart (OU spend trends month over month)

📊 Quarterly Trends → Bar chart (Service spend trends per quarter)

🥧 Yearly Trends → Pie chart (OU spend distribution per year)

This means execs never need to touch Athena — they can toggle between:

Daily breakdown (Top OUs, Services)

Monthly trends

Quarterly trends

Yearly roll-ups