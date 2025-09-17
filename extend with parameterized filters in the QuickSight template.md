👉 Do you also want me to extend this with parameterized filters in the QuickSight template (e.g., filter by OU, Account, or Service dynamically)?

Awesome 🎯 — let’s extend the QuickSight OU Dashboard template with parameterized filters so you (and your FinOps team) can interactively filter spend by OU, Account, or Service.

🔹 1. Add Parameters for Filters

QuickSight supports parameters + controls so you can dynamically filter.
We’ll add:

OUFilter → filter by Organizational Unit

AccountFilter → filter by Account

ServiceFilter → filter by AWS Service

🔹 2. Updated Template JSON (focus-ou-template.json)

Here’s a trimmed version showing parameterized filters added on top of visuals:

{
  "AwsAccountId": "123456789012",
  "TemplateId": "focus-ou-template",
  "Name": "FOCUS OU Template",
  "SourceEntity": {
    "SourceAnalysis": {
      "Arn": "arn:aws:quicksight:us-east-1:123456789012:analysis/focus-ou-analysis",
      "DataSetReferences": [
        {
          "DataSetArn": "arn:aws:quicksight:us-east-1:123456789012:dataset/focus-ou-dataset",
          "DataSetPlaceholder": "FOCUS_OU"
        }
      ]
    }
  },
  "Version": {
    "Description": "OU Dashboard v2 with Filters",
    "Sheets": [
      {
        "Name": "OU Overview",
        "ParameterControls": [
          {
            "Dropdown": {
              "ParameterControlId": "OUFilterControl",
              "Title": "Select OU",
              "SourceParameterName": "OUFilter",
              "SelectableValues": {
                "LinkToDataSetColumn": {
                  "DataSetIdentifier": "FOCUS_OU",
                  "ColumnName": "OU"
                }
              }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "AccountFilterControl",
              "Title": "Select Account",
              "SourceParameterName": "AccountFilter",
              "SelectableValues": {
                "LinkToDataSetColumn": {
                  "DataSetIdentifier": "FOCUS_OU",
                  "ColumnName": "AccountName"
                }
              }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "ServiceFilterControl",
              "Title": "Select Service",
              "SourceParameterName": "ServiceFilter",
              "SelectableValues": {
                "LinkToDataSetColumn": {
                  "DataSetIdentifier": "FOCUS_OU",
                  "ColumnName": "serviceCode"
                }
              }
            }
          }
        ],
        "Visuals": [
          {
            "BarChartVisual": {
              "Title": { "Text": "Spend by OU" },
              "ConditionalFormatting": {},
              "ChartConfiguration": {
                "FieldWells": {
                  "BarChartAggregatedFieldWells": {
                    "Category": [
                      { "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }
                    ],
                    "Values": [
                      {
                        "FieldId": "effectiveCost",
                        "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" },
                        "AggregationFunction": { "SimpleNumericalAggregation": "SUM" }
                      }
                    ]
                  }
                },
                "Actions": [],
                "FilterGroups": [
                  {
                    "FilterGroupId": "OUFilterGroup",
                    "Filters": [
                      {
                        "CategoryFilter": {
                          "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" },
                          "Configuration": { "CustomFilterListConfiguration": { "MatchOperator": "EQUALS", "ParameterName": "OUFilter" } }
                        }
                      }
                    ]
                  },
                  {
                    "FilterGroupId": "AccountFilterGroup",
                    "Filters": [
                      {
                        "CategoryFilter": {
                          "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "AccountName" },
                          "Configuration": { "CustomFilterListConfiguration": { "MatchOperator": "EQUALS", "ParameterName": "AccountFilter" } }
                        }
                      }
                    ]
                  },
                  {
                    "FilterGroupId": "ServiceFilterGroup",
                    "Filters": [
                      {
                        "CategoryFilter": {
                          "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "serviceCode" },
                          "Configuration": { "CustomFilterListConfiguration": { "MatchOperator": "EQUALS", "ParameterName": "ServiceFilter" } }
                        }
                      }
                    ]
                  }
                ]
              }
            }
          }
        ]
      }
    ],
    "Parameters": {
      "OUFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
      "AccountFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
      "ServiceFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } }
    }
  }
}


3. Terraform Integration

Update your Terraform module’s QuickSight section:

resource "aws_quicksight_template" "focus_ou" {
  template_id    = "focus-ou-template"
  name           = "FOCUS OU Template"
  aws_account_id = var.account_id

  source_entity {
    source_analysis {
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:analysis/focus-ou-analysis"
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }
}

resource "aws_quicksight_dashboard" "focus_ou" {
  dashboard_id   = "focus-ou-dashboard"
  name           = "FOCUS OU Dashboard"
  aws_account_id = var.account_id

  source_entity {
    source_template {
      arn = aws_quicksight_template.focus_ou.arn
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }
}


✅ End Result

QuickSight OU Dashboard now includes interactive dropdowns:

Filter by OU

Filter by Account

Filter by Service

All visuals respond to these filters dynamically.

Team can explore cost trends, service breakdowns, and discounts per OU without editing queries.

👉 Do you want me to also add a time range filter (Year/Month) so users can drill into specific months or quarters directly from the dashboard?


Perfect ✅ — let’s extend the FOCUS OU QuickSight dashboard template to include a time range filter so users can drill into specific years, months, or quarters directly.

🔹 1. Add Time Parameters

We’ll add two new parameters:

YearFilter → select the billing year

MonthFilter → select the billing month

Later, we can combine them for a Year-Month range selector.

🔹 2. Updated Template JSON (Time Filters Added)

Here’s the enhanced template JSON snippet with OU, Account, Service, Year, Month filters:

{
  "AwsAccountId": "123456789012",
  "TemplateId": "focus-ou-template",
  "Name": "FOCUS OU Template",
  "Version": {
    "Description": "OU Dashboard v3 with Time Filters",
    "Sheets": [
      {
        "Name": "OU Overview",
        "ParameterControls": [
          {
            "Dropdown": {
              "ParameterControlId": "OUFilterControl",
              "Title": "Select OU",
              "SourceParameterName": "OUFilter",
              "SelectableValues": { "LinkToDataSetColumn": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "AccountFilterControl",
              "Title": "Select Account",
              "SourceParameterName": "AccountFilter",
              "SelectableValues": { "LinkToDataSetColumn": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "AccountName" } }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "ServiceFilterControl",
              "Title": "Select Service",
              "SourceParameterName": "ServiceFilter",
              "SelectableValues": { "LinkToDataSetColumn": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "serviceCode" } }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "YearFilterControl",
              "Title": "Select Year",
              "SourceParameterName": "YearFilter",
              "SelectableValues": { "LinkToDataSetColumn": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "year" } }
            }
          },
          {
            "Dropdown": {
              "ParameterControlId": "MonthFilterControl",
              "Title": "Select Month",
              "SourceParameterName": "MonthFilter",
              "SelectableValues": { "LinkToDataSetColumn": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "month" } }
            }
          }
        ],
        "Visuals": [
          {
            "LineChartVisual": {
              "Title": { "Text": "Spend Trend by OU" },
              "ChartConfiguration": {
                "FieldWells": {
                  "LineChartAggregatedFieldWells": {
                    "Category": [{ "FieldId": "month", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "month" } }],
                    "Colors": [{ "FieldId": "OU", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "OU" } }],
                    "Values": [{ "FieldId": "effectiveCost", "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "effectiveCost" }, "AggregationFunction": { "SimpleNumericalAggregation": "SUM" } }]
                  }
                },
                "FilterGroups": [
                  {
                    "FilterGroupId": "YearFilterGroup",
                    "Filters": [
                      {
                        "CategoryFilter": {
                          "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "year" },
                          "Configuration": { "CustomFilterListConfiguration": { "MatchOperator": "EQUALS", "ParameterName": "YearFilter" } }
                        }
                      }
                    ]
                  },
                  {
                    "FilterGroupId": "MonthFilterGroup",
                    "Filters": [
                      {
                        "CategoryFilter": {
                          "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "month" },
                          "Configuration": { "CustomFilterListConfiguration": { "MatchOperator": "EQUALS", "ParameterName": "MonthFilter" } }
                        }
                      }
                    ]
                  }
                ]
              }
            }
          }
        ]
      }
    ],
    "Parameters": {
      "OUFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
      "AccountFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
      "ServiceFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
      "YearFilter": { "IntegerParameter": { "DefaultValues": { "StaticValues": [2025] } } },
      "MonthFilter": { "IntegerParameter": { "DefaultValues": { "StaticValues": [1] } } }
    }
  }
}


🔹 3. Terraform Integration (No Change in Structure)

Terraform just needs to point to the updated template JSON.

resource "aws_quicksight_template" "focus_ou" {
  template_id    = "focus-ou-template"
  name           = "FOCUS OU Template"
  aws_account_id = var.account_id

  source_entity {
    source_analysis {
      arn = "arn:aws:quicksight:${var.region}:${var.account_id}:analysis/focus-ou-analysis"
      data_set_references = [
        {
          data_set_arn       = aws_quicksight_data_set.focus_ou.arn
          data_set_placeholder = "FOCUS_OU"
        }
      ]
    }
  }
}


✅ End Result

Dashboard now has interactive dropdowns for:

OU

Account

Service

Year

Month

Analysts can slice costs by OU → Account → Service → Time.

Makes it easy to drill into specific months or quarters for trend analysis.


👉 Do you want me to also add a date range slider (between start & end month) instead of just dropdowns, so you can do rolling-window analysis (e.g., last 6 months)?

Great choice 🎯 — let’s take this further by adding a date range slider so you can do rolling-window analysis (e.g., last 3 months, last 6 months, year-to-date).

🔹 1. Approach

Instead of just filtering by single Year/Month, we’ll add two new date parameters:

StartDate → beginning of the range

EndDate → end of the range

These will connect to a range slider in QuickSight so analysts can interactively adjust the billing window.

⚠️ Since your FOCUS dataset stores year + month (not a native date), we’ll create a synthetic date column in Athena using:


CAST(DATE_PARSE(CONCAT(year, '-', month, '-01'), '%Y-%m-%d') AS DATE) AS billing_date


This way, QuickSight can handle date filters natively.

🔹 2. Updated Athena SQL (FOCUS OU Dataset)

Update the dataset SQL in Terraform:

SELECT f.year,
       f.month,
       CAST(DATE_PARSE(CONCAT(f.year, '-', f.month, '-01'), '%Y-%m-%d') AS DATE) AS billing_date,
       f.usageAccountId,
       m.AccountName,
       m.OU,
       f.serviceCode,
       f.region,
       f.effectiveCost,
       f.listCost,
       f.billedCost,
       f.usageAmount,
       f.unit
FROM default.focus_org_billing f
LEFT JOIN default.account_ou_mapping m
  ON f.usageAccountId = m.AccountId

🔹 3. Template JSON with Date Range Slider

Here’s the new JSON snippet with a range slider filter for billing_date:

{
  "Parameters": {
    "StartDate": { "DateTimeParameter": { "DefaultValues": { "StaticValues": ["2025-01-01"] } } },
    "EndDate": { "DateTimeParameter": { "DefaultValues": { "StaticValues": ["2025-12-31"] } } },
    "OUFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
    "AccountFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } },
    "ServiceFilter": { "StringParameter": { "DefaultValues": { "StaticValues": ["ALL"] } } }
  },
  "Sheets": [
    {
      "Name": "OU Overview",
      "ParameterControls": [
        {
          "DateTimePicker": {
            "ParameterControlId": "StartDateControl",
            "Title": "Start Date",
            "SourceParameterName": "StartDate"
          }
        },
        {
          "DateTimePicker": {
            "ParameterControlId": "EndDateControl",
            "Title": "End Date",
            "SourceParameterName": "EndDate"
          }
        }
      ],
      "FilterGroups": [
        {
          "FilterGroupId": "DateRangeFilter",
          "Filters": [
            {
              "RelativeDatesFilter": {
                "Column": { "DataSetIdentifier": "FOCUS_OU", "ColumnName": "billing_date" },
                "AnchorDateConfiguration": { "ParameterName": "StartDate" },
                "MinimumGranularity": "DAY",
                "NullOption": "NON_NULLS_ONLY",
                "TimeGranularity": "DAY",
                "RangeMinimumValue": { "ParameterName": "StartDate" },
                "RangeMaximumValue": { "ParameterName": "EndDate" }
              }
            }
          ]
        }
      ]
    }
  ]
}


🔹 4. Terraform Integration

Nothing changes in structure — just point Terraform to the updated SQL + template JSON:

resource "aws_quicksight_data_set" "focus_ou" {
  data_set_id   = "focus-ou-dataset"
  name          = "FOCUS_OU"
  aws_account_id = var.account_id
  import_mode    = "DIRECT_QUERY"

  physical_table_map = {
    "FOCUS_OU" = {
      custom_sql = {
        data_source_arn = aws_quicksight_data_source.athena.arn
        name            = "FOCUS_OU_SQL"
        sql_query       = file("${path.module}/sql/focus-ou.sql") # <-- updated with billing_date
      }
    }
  }
}


✅ End Result

Your dashboard will now include:

OU Filter (dropdown)

Account Filter (dropdown)

Service Filter (dropdown)

Date Range Slider (start + end date picker)

🔎 Users can do rolling-window cost analysis:

Last 3 months vs. last 6 months

Fiscal quarters

Year-to-date