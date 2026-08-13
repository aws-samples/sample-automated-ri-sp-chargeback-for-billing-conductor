# Automated RI/SP Chargeback with AWS Billing Conductor

This solution deploys an automated pipeline that calculates the exact dollar savings each member account received from management-held Reserved Instances (RIs) and Savings Plans (SPs), then injects those savings as proportional credit adjustments into AWS Billing Conductor (ABC). The result is an accurate, auditable chargeback number for every account, every month — fully automated.

## Overview

Organizations that centrally purchase AWS Reserved Instances and Savings Plans can save 30–72% on compute and database costs compared to on-demand pricing. However, when you use ABC to produce pro forma billing, commitments purchased at the management account level sit outside all ABC billing groups. This means member account pro forma bills reflect only on-demand equivalent costs — discounts are absorbed silently.

This solution bridges that gap using three layers:

- **Data layer** — AWS Cost and Usage Report (CUR) 2.0, cataloged with AWS Glue and queried through Amazon Athena, provides the ground truth on exactly how much each member account saved from management-held RIs and SPs.
- **Calculation layer** — An Athena SQL query identifies all CUR 2.0 line items where usage was covered by RI or SP discounts (`SavingsPlanCoveredUsage` and `DiscountedUsage`), then calculates the exact dollar savings per member account by computing the difference between on-demand cost and the discounted effective cost.
- **Injection layer** — AWS Lambda reads those savings amounts and creates flat Custom Line Items (credits) in ABC for each member account, reducing their pro forma bill by exactly the discount they received.

> **Scope:** This solution is designed for organizations where commitments are purchased exclusively at the management level and member accounts do not own their own RIs or SPs. The management account must be positioned outside all ABC billing groups.

## Architecture

The CloudFormation template provisions the following resources:

| Resource | Description |
|---|---|
| `BillingGroupMappingFunction` (Lambda 1) | Builds the account-to-billing-group map from ABC and stores it in SSM Parameter Store |
| `ApplyRISPCreditsFunction` (Lambda 2) | Runs the Athena savings query, calculates per-account credits, and creates Custom Line Items in ABC |
| `BillingGroupMappingRole` | Least-privilege IAM role for Lambda 1 (Billing Conductor read, SSM write) |
| `ApplyRISPCreditsRole` | Least-privilege IAM role for Lambda 2 (Billing Conductor write, Athena, S3, SSM read) |
| `AuditBucket` | Encrypted S3 bucket for storing per-billing-period audit logs (AES-256, 365-day lifecycle) |
| `AccountBillingGroupMapParam` | SSM Parameter Store entry holding the account-to-billing-group JSON map |
| `MonthlyChargebackSchedule` | EventBridge rule triggering the pipeline on the 5th of each month at 8:00 AM UTC (disabled by default) |

The pipeline executes in two ordered stages:

1. **Lambda 1** — Calls the Billing Conductor API to list all billing groups and their associated member accounts, excludes the management account, and stores the resulting `{account_id: billing_group_arn}` map in SSM.
2. **Lambda 2** — Runs the Athena savings query, loads the SSM account map, creates one Custom Line Item credit per discount type (SP and RI) per account, validates that the total credits applied match Athena totals within USD 1.00, and writes a full audit log to S3.

## Prerequisites

Before deploying, confirm the following are in place:

1. **CUR 2.0 data export configured** — AWS Data Exports (CUR 2.0) must be enabled on the management account with daily granularity, resource IDs included, and Parquet output format to S3. See [Creating a standard data export](https://docs.aws.amazon.com/cur/latest/userguide/dataexports-create-standard.html).

2. **AWS Glue crawler cataloging CUR data** — A Glue crawler must catalog the CUR 2.0 S3 prefix into the AWS Glue Data Catalog so Athena can query it. See [Adding an AWS Glue crawler](https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html).

   After the crawler runs, verify your Athena database and table are accessible by running the following in the Athena console:
   ```sql
   SHOW COLUMNS FROM your_database.your_table;
   ```

   The following CUR 2.0 columns are required:

   | Column | Purpose |
   |---|---|
   | `line_item_line_item_type` | Filter for `SavingsPlanCoveredUsage` and `DiscountedUsage` |
   | `line_item_usage_account_id` | Identify the consuming member account |
   | `pricing_public_on_demand_cost` | On-demand equivalent cost |
   | `savings_plan_savings_plan_effective_cost` | Actual cost after SP discount |
   | `reservation_effective_cost` | Actual cost after RI discount |
   | `savings_plan_savings_plan_a_r_n` | Identifies which Savings Plan provided the discount |
   | `reservation_reservation_a_r_n` | Identifies which RI provided the discount |

3. **ABC billing groups configured** — Member accounts must already be assigned to billing groups in ABC. Accounts within a billing group must not own their own RIs or SPs.

## Deployment

### Option A: AWS Management Console

1. Download `RI_SP_Chargeback.yaml` from this repository.
2. Open the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation).
3. Choose **Create stack** → **With new resources (standard)**.
4. Under **Specify template**, choose **Upload a template file**, select the YAML file, and choose **Next**.
5. Set the stack name to `risp-chargeback-pipeline` and fill in the parameters (see table below).
6. On the **Configure stack options** page, keep defaults and choose **Next**.
7. On the **Review** page, check **I acknowledge that AWS CloudFormation might create IAM resources with custom names**, then choose **Submit**.
8. Wait for the stack status to show `CREATE_COMPLETE` (approximately 2–3 minutes).

### Option B: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name risp-chargeback-pipeline \
  --template-body file://RI_SP_Chargeback.yaml \
  --parameters \
    ParameterKey=PayerAccountId,ParameterValue=123456789012 \
    ParameterKey=AthenaDatabase,ParameterValue=cur2 \
    ParameterKey=AthenaTable,ParameterValue=data \
    ParameterKey=CURBucketName,ParameterValue=your-cur-bucket-name \
    ParameterKey=BillingPeriod,ParameterValue="2026-02:" \
    ParameterKey=DryRun,ParameterValue=true \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Monitor progress:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name risp-chargeback-pipeline
```

## Template Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `PayerAccountId` | String | *(required)* | Your 12-digit AWS management account ID |
| `AthenaDatabase` | String | `cur2` | Glue database containing your CUR 2.0 table |
| `AthenaTable` | String | `data` | Table name within the Athena database |
| `CURBucketName` | String | *(required)* | S3 bucket where CUR 2.0 data is stored |
| `BillingPeriod` | String | `2026-02:` | Billing period in CUR 2.0 partition format (`YYYY-MM:` with trailing colon) |
| `DryRun` | String | `true` | Set to `true` for simulation mode — no credits are created |
| `EnableSchedule` | String | `false` | Set to `true` to enable the monthly EventBridge schedule |

> **Note:** The `BillingPeriod` parameter uses the CUR 2.0 partition format (for example, `2026-06:`). Update this value each month, or enable the EventBridge schedule for automatic monthly processing.

## How the Savings Calculation Works

The core formula is:

```
Savings = pricing_public_on_demand_cost − effective_cost
```

Lambda 2 embeds the following Athena SQL query and runs it automatically when invoked. It calculates combined SP and RI savings per member account for a given billing period in a single pass:

```sql
SELECT
    line_item_usage_account_id AS account_id,
    ROUND(SUM(CASE
        WHEN savings_plan_savings_plan_a_r_n <> ''
        THEN CAST(pricing_public_on_demand_cost AS double)
           - CAST(savings_plan_savings_plan_effective_cost AS double)
        ELSE 0 END), 4) AS sp_savings_credit,
    ROUND(SUM(CASE
        WHEN reservation_reservation_a_r_n <> ''
        THEN CAST(pricing_public_on_demand_cost AS double)
           - CAST(reservation_effective_cost AS double)
        ELSE 0 END), 4) AS ri_savings_credit,
    ROUND(SUM(CASE
        WHEN savings_plan_savings_plan_a_r_n <> ''
        THEN CAST(pricing_public_on_demand_cost AS double)
           - CAST(savings_plan_savings_plan_effective_cost AS double)
        WHEN reservation_reservation_a_r_n <> ''
        THEN CAST(pricing_public_on_demand_cost AS double)
           - CAST(reservation_effective_cost AS double)
        ELSE 0 END), 4) AS total_discount_credit
FROM ${AthenaDatabase}.${AthenaTable}
WHERE billing_period = '${BillingPeriod}'
  AND line_item_line_item_type IN ('SavingsPlanCoveredUsage', 'DiscountedUsage')
GROUP BY line_item_usage_account_id
HAVING SUM(CASE
    WHEN savings_plan_savings_plan_a_r_n <> ''
    THEN CAST(pricing_public_on_demand_cost AS double)
       - CAST(savings_plan_savings_plan_effective_cost AS double)
    WHEN reservation_reservation_a_r_n <> ''
    THEN CAST(pricing_public_on_demand_cost AS double)
       - CAST(reservation_effective_cost AS double)
    ELSE 0 END) > 0
ORDER BY total_discount_credit DESC
```

- `sp_savings_credit` — SP-covered usage rows; covers Compute, EC2 Instance, SageMaker AI, and Database SP types.
- `ri_savings_credit` — RI-discounted rows; covers EC2, RDS, Redshift, ElastiCache, and OpenSearch.
- `total_discount_credit` — Combined SP and RI savings per account.
- `HAVING` clause — Filters out accounts with zero or negative total savings.
- `ROUND(..., 4)` — Rounds to 4 decimal places for precision.

> **Tip:** You can run this query directly in the Athena console to manually verify savings numbers before running the full pipeline.

## Validate with Dry Run

The stack deploys with `DRY_RUN=true` by default. No Custom Line Items are created until you explicitly disable dry-run mode.

### Step 1: Run Lambda 1 to build the account map

1. Open the [AWS Lambda console](https://console.aws.amazon.com/lambda).
2. Navigate to `BillingGroupMappingFunction`.
3. Choose **Test**, use the default test event (`{}`), and choose **Invoke**.
4. Verify in the response that your accounts are correctly mapped to billing groups.

### Step 2: Run Lambda 2 in dry-run mode

1. Navigate to `ApplyRISPCreditsFunction` in the Lambda console.
2. Choose **Test** and provide the following test event:

   ```json
   {
     "billing_period": "2026-06"
   }
   ```

3. Choose **Invoke** and review the response. A successful dry-run response looks like:

   ```json
   {
     "billing_period": "2026-06",
     "dry_run": true,
     "dry_run_simulated": 42,
     "skipped": 1,
     "failed": 0,
     "validation": {
       "total_athena_savings": 18450.00,
       "total_credits_applied": 0.0,
       "message": "DRY RUN - no credits applied. Review simulated output."
     }
   }
   ```

### Step 3: Verify dry-run results

Confirm the following before going live:

- All expected accounts appear in the output
- Credit amounts align with your CUR data expectations
- No `failed` entries in the response
- The `total_athena_savings` value looks reasonable for your organization's RI/SP portfolio

## Go Live and Automate

After validating dry-run output, disable dry-run mode and enable the monthly schedule.

### Disable dry-run mode

```bash
aws lambda update-function-configuration \
  --function-name ApplyRISPCreditsFunction \
  --environment "Variables={
    PAYER_ACCOUNT_ID=123456789012,
    SSM_PARAM_NAME=/chargeback/account-billing-group-map,
    AUDIT_BUCKET=risp-chargeback-audit-123456789012-us-east-1,
    DRY_RUN=false,
    ATHENA_DATABASE=cur2,
    ATHENA_TABLE=data,
    ATHENA_OUTPUT_LOCATION=s3://risp-chargeback-audit-123456789012-us-east-1/athena-results/
  }" \
  --region us-east-1
```

Or update directly in the Lambda console: **Configuration → Environment variables → set `DRY_RUN` to `false`**.

### Enable the monthly EventBridge schedule

The EventBridge rule triggers on the 5th of each month at 8:00 AM UTC, allowing CUR 2.0 data to finalize (typically Day 3–4) before processing.

```bash
aws events enable-rule \
  --name risp-chargeback-monthly-schedule \
  --region us-east-1
```

Or set `EnableSchedule=true` when deploying or updating the stack.

> **Tip:** For production environments, consider chaining Lambda 1 → Lambda 2 using AWS Step Functions for robust error handling and retry logic.

## Before and After

| Metric | Before (ABC default) | After (with chargeback credits) |
|---|---|---|
| Member account pro forma bill | On-demand equivalent only | On-demand minus proportional RI/SP credit |
| Management visibility into distribution | None — discounts absorbed silently | Full audit trail in S3 per billing period |
| Chargeback accuracy | Overstated (does not reflect actual cost) | Accurate to within USD 1.00 of actual effective cost |
| Manual effort per month | N/A (no mechanism existed) | Zero — fully automated pipeline |

## Cost Estimate

This solution incurs minimal ongoing costs:

| Service | Monthly Estimate | Notes |
|---|---|---|
| AWS Lambda | < USD 0.01 | ~2 invocations/month, each <30 seconds |
| Amazon Athena | Free tier + USD 0.01–5.00 | Depends on CUR data volume (USD 5/TB scanned) |
| Amazon S3 (audit logs) | < USD 0.01 | ~1 KB per account per month |
| SSM Parameter Store | Free | Standard parameters at no charge |
| Amazon EventBridge | Free | 1 rule, 1 invocation/month (free tier) |

**Estimated total: less than USD 5/month** for most organizations. The primary cost driver is Athena query scans — using Parquet format (the CUR 2.0 default) significantly reduces scan volume.

## Limitations and Considerations

- **Scenario scope:** This solution does not support member-account-owned commitments (see Scope above).
- **Single billing period:** Each pipeline execution processes one billing period. Custom Line Items can only be backdated to the immediately preceding billing month.
- **Credit granularity:** Credits are applied at the account level, not at the resource or service level. To get service-level breakdown, extend the Athena query to include `product_servicecode` grouping.
- **CUR data timing:** CUR 2.0 data may take 24–48 hours to finalize after month-end. The EventBridge schedule targets the 5th of the month to allow sufficient processing time.
- **Rounding:** With many accounts, the sum of individual credits may differ from total savings by up to USD 1.00 due to floating-point rounding. The validation step flags this if exceeded.

## Cleaning Up

To avoid ongoing charges, delete the CloudFormation stack:

```bash
aws cloudformation delete-stack \
  --stack-name risp-chargeback-pipeline \
  --region us-east-1
```

> **Note:** Custom Line Items already created in ABC are not deleted when you remove the stack — they remain in the billing period they were created for. The S3 audit bucket has a `DeletionPolicy` of `Retain` by default to preserve audit logs. You must empty and delete the bucket manually if you want to remove all resources.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
