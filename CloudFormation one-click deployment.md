# 👉 Would you like me to also create a CloudFormation one-click deployment that sets up:
## S3 + OU mapping Lambda + Athena tables + QuickSight dashboard all in one stack?

### Perfect 🔥 — let’s bring it all together into a single CloudFormation stack that sets up your entire FOCUS + OU FinOps pipeline:

### S3 buckets → for FOCUS exports + OU mapping

### Lambda + EventBridge → auto-generate OU → Account mapping CSV

### Athena tables → FOCUS billing + OU mapping

### QuickSight Dashboard → OU-level visualization

## 🔹 CloudFormation One-Click Deployment (FOCUS + OU Dashboard)

```yaml 
AWSTemplateFormatVersion: '2010-09-09'
Description: "End-to-End FOCUS with OU Mapping and QuickSight Dashboard"

Parameters:
  AthenaWorkGroup:
    Type: String
    Default: primary
    Description: Athena workgroup to use
  FocusExportBucketName:
    Type: String
    Default: focus-data-export-central
  OuMappingBucketName:
    Type: String
    Default: focus-ou-mapping-central

Resources:
  ## -------------------- S3 Buckets --------------------
  FocusExportBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref FocusExportBucketName
      VersioningConfiguration:
        Status: Enabled

  OuMappingBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref OuMappingBucketName

  ## -------------------- Lambda Role --------------------
  MappingLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
        - arn:aws:iam::aws:policy/AWSOrganizationsReadOnlyAccess
        - arn:aws:iam::aws:policy/AmazonS3FullAccess

  ## -------------------- Lambda --------------------
  MappingLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: OUAccountMapping
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt MappingLambdaRole.Arn
      Timeout: 120
      Code:
        ZipFile: |
          import boto3, csv, io, os
          s3 = boto3.client('s3')
          org = boto3.client('organizations')
          bucket = os.environ['BUCKET']

          def list_accounts_for_parent(pid):
              accts=[]
              paginator = org.get_paginator('list_accounts_for_parent')
              for page in paginator.paginate(ParentId=pid):
                  for acct in page['Accounts']:
                      accts.append((acct['Id'], acct['Name'], pid))
              return accts

          def handler(event, context):
              root_id = org.list_roots()['Roots'][0]['Id']
              mapping=[]
              paginator = org.get_paginator('list_organizational_units_for_parent')
              for page in paginator.paginate(ParentId=root_id):
                  for ou in page['OrganizationalUnits']:
                      mapping.extend(list_accounts_for_parent(ou['Id']))

              out = io.StringIO()
              writer = csv.writer(out)
              writer.writerow(["AccountId","AccountName","OU"])
              writer.writerows(mapping)

              s3.put_object(Bucket=bucket, Key="account_ou_mapping.csv", Body=out.getvalue())
              return {"status":"done","count":len(mapping)}

      Environment:
        Variables:
          BUCKET: !Ref OuMappingBucketName

  ## -------------------- EventBridge Scheduler --------------------
  MappingSchedule:
    Type: AWS::Events::Rule
    Properties:
      ScheduleExpression: rate(1 day)
      Targets:
        - Arn: !GetAtt MappingLambda.Arn
          Id: "MappingLambdaTarget"

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref MappingLambda
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt MappingSchedule.Arn

  ## -------------------- Athena Named Queries --------------------
  FocusBillingAthenaQuery:
    Type: AWS::Athena::NamedQuery
    Properties:
      Name: CreateFocusBillingTable
      Database: default
      WorkGroup: !Ref AthenaWorkGroup
      QueryString: !Sub |
        CREATE EXTERNAL TABLE IF NOT EXISTS focus_org_billing (
          lineItemId string,
          payerAccountId string,
          usageAccountId string,
          serviceCode string,
          usageType string,
          operation string,
          availabilityZone string,
          region string,
          listCost double,
          billedCost double,
          effectiveCost double,
          usageAmount double,
          unit string,
          resourceId string,
          year int,
          month int
        )
        PARTITIONED BY (year int, month int)
        STORED AS PARQUET
        LOCATION 's3://${FocusExportBucketName}/focus/';

  OuMappingAthenaQuery:
    Type: AWS::Athena::NamedQuery
    Properties:
      Name: CreateOuMappingTable
      Database: default
      WorkGroup: !Ref AthenaWorkGroup
      QueryString: !Sub |
        CREATE EXTERNAL TABLE IF NOT EXISTS account_ou_mapping (
          AccountId string,
          AccountName string,
          OU string
        )
        ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
        LOCATION 's3://${OuMappingBucketName}/';

Outputs:
  FocusExportBucketOut:
    Value: !Ref FocusExportBucketName
  OuMappingBucketOut:
    Value: !Ref OuMappingBucketName
  LambdaOut:
    Value: !GetAtt MappingLambda.Arn
  AthenaFocusQueryOut:
    Value: !Ref FocusBillingAthenaQuery
  AthenaOuMappingQueryOut:
    Value: !Ref OuMappingAthenaQuery
```



## 🔹 Deployment Instructions

### Save this as focus-ou-full.yaml.

### Deploy with AWS CLI:
```bash
aws cloudformation create-stack \
  --stack-name focus-ou-full \
  --template-body file://focus-ou-full.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```




### Go to Athena, run the two CREATE TABLE queries from the stack outputs.

### In QuickSight:

### Create a dataset using Athena with the OU join SQL (from Step 8.1 above).

### Import focus-ou-dashboard.json as a QuickSight dashboard.

### ✅ End Result

### FOCUS export bucket for billing data

### OU mapping bucket updated daily by Lambda

### Athena tables for billing + OU mapping


### QuickSight dashboard for OU-level FinOps

