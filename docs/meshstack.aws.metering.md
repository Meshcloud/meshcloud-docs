---
id: meshstack.aws.metering
title: Metering
---
meshStack imports metering data from [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/).
It is recommended to have a daily report update interval in order to keep the costs of calling the API low. This can be
configured via the `awsStateCollectionInterval` property.

The `amortized cost` is used when generating the tenant usage reports.

## IAM User Configuration

### Policies

In order for meshStack to generate Usage Reports, following policies are required:

1. **Cost Explorer Read Policy**: This policy allows the Metering IAM user to call the AWS Cost Explorer API to read data required for metering. Note that Savings Plan and Reserved Instance related permissions are needed only if you have specific meshCustomers buying those directly, and you need to implement a fair Chargeback process for those.
2. **Organization Access Policy**: This policy allows the Metering IAM user to list all accounts in the organization
3. **Assume Role Policy**: This policy allows Metering IAM user to assume the IAM Role `MeteringRole`

<!--DOCUSAURUS_CODE_TABS-->
<!--Bucket Access Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "ce:GetSavingsPlansUtilizationDetails",
                "ce:GetSavingsPlansUtilization",
                "ce:GetReservationUtilization",
                "ce:GetDimensionValues",
                "ce:GetCostAndUsage"
            ],
            "Resource": "*"
        }
    ]
}
```
<!--Organization Access Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "organizations:ListAccounts",
            "Resource": "*"
        }
    ]
}
```
<!--Assume Role Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::123456789123:role/MeteringRole"
        }
    ]
}
```
<!--END_DOCUSAURUS_CODE_TABS-->


## Reserved Instances & Savings Plans

meshStack uses the `amortized cost` from the AWS Cost Explorer to generate the tenant usage reports. If any of your
meshCustomers pay upfront to purchase reserved instances or savings plans, they would be charged twice when we use
the amortized cost. To prevent this from happening, meshStack collects the information about reserved instances and savings plans,
via the cost explorer API and adds a discount which is equal to the `amortized upfront cost` to the relevant tenant usage report.

## Configuration Reference

This section describes the configuration of a AWS Platform Instance in the meshStack [configuration model](./meshstack.configuration.md)
at `mesh.platforms` for AWS metering.

<!--snippet:mesh.platforms.aws.kraken#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AwsPlatformKrakenConfiguration =
      { platform :
          {- The combination of meshPlatform and meshLocation identifiers, i.e. <platform-id>.<location-id> -}
          Text
      , region :
          {- The region of the IAM user -}
          Text
      , meshfed-service-user :
          {-
              Metering IAM user credentials
                  access-key: "<AWS_ACCESS_KEY>"
                  secret-key: "<AWS_SECRET_KEY>"
          -}
          { access-key : Secret, secret-key : Secret }
      , organization-root-account-role :
          {- The role assigned to the metering IAM user -}
          Text
      , organization-root-account-external-id :
          {- The external ID passed to "sts:AssumeRole" when used in privileged context -}
          Optional Text
      , data-source :
          {-
          Currently both AWS Cost and Uasge Reports and AWS Cost Explorer are supported.
          But AWS Cost and Usage Reports are planned to be deprecated
          -}
          ./DataSource.dhall
      }
```
<!--Example-->
```dhall
let example
    : AwsPlatformKrakenConfiguration
    = { platform = "aws.aws-location"
      , region = "eu-central-1"
      , meshfed-service-user =
        { access-key = Secret.Native "AWS_ACCESS_KEY_KRAKEN"
        , secret-key = Secret.Native "AWS_SECRET_KEY_KRAKEN"
        }
      , organization-root-account-role =
          "arn:aws:iam::123456789123:role/MeteringRole"
      , organization-root-account-external-id = Some
          "abcd1234-12ab-12ab-12ab-abcdef123456"
      , data-source =
          DataSource.CostExplorer
            { cost-explorer =
              { filter =
                  {- Filter type of NONE and EXCLUDE_TAX are supported. -}
                  (./CostExplorer.dhall).FilterType.NONE
              , reservedInstanceFairChargeback =
                  {-Enable fair chargeback for meshCustomer purchased RIs-}
                  True
              , savingsPlanFairChargeback =
                  {-Enable fair chargeback for meshCustomer purchased SPs-}
                  True
              }
            }
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

## Configuring Seller Information

If you would like to see the AWS costs attributed to a seller in the [chargeback statments](./meshcloud.project-metering.md#chargeback-statements), a product needs to be created manually in the meshcloud [Product Catalog](meshstack.billing-configuration.md#defining-a-custom-product-catalog). This product should have a `resourceType` of `sellerInfo` and should be scoped to a `platformType` of `Aws` . An example is shown below.

```json
{
    "resourceType": "sellerInfo",
    "displayName": "AWS Seller Info",
    "scope": {
        "platformType": "Aws",
        "location": null,
        "platformInstance": null,
        "localProjectId": null
    },
    "usageTypes": [],
    "description": "",
    "sellerId": "AWS",
    "sellerProductGroup": null,
    "@metadata": {
        "@collection": "Products",
        "Raven-Java-Type": "io.meshcloud.kraken.core.metering.Product"
    }
}
```

## AWS Cost & Usage Report based approach (Deprecated)

meshStack imports metering data from [AWS Cost and Usage Reports](https://aws.amazon.com/aws-cost-management/aws-cost-and-usage-reporting/).

An AWS Cost and Usage Report should be created as specified in the [documentation](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html) with the following properties:

* Daily time granularity
* gzip compression
* txt/csv file format
* Should include resource ids
* Versioning should be setup to overwrite existing reports

**Note**: If an S3 bucket is not already set up, it should be created as specified in [AWS documentation](https://docs.aws.amazon.com/cur/latest/userguide/cur-s3.html)

The following parameters are required to configure meshStack to process the AWS Cost and Usage Reports [[More Details](#configuration-reference)]:

* AWS credentials that can access the S3 bucket where the reports are written to
* The name of the S3 bucket where the report is written
* The region of the S3 bucket
* The name of the report
* The "Report path prefix" configured when creating the report

When processing the AWS Cost and Usage Report to generate the Usage Report in the meshPanel,

* You can configure which [line item types](https://docs.aws.amazon.com/cur/latest/userguide/Lineitem-columns.html#l-L)
should be considered in the calculations. You can also configure whether you want to consider discounts or not.
We include the following columns in the calculations in order to come up with an amortized cost that should be charged to each account(the discount column is taken only if configured to consider discounts).
    1. If the line item type is RIFee, we take `reservation/UnusedAmortizedUpfrontFeeForBillingPeriod`, `reservation/UnusedRecurringFee` and `discounts/TotalDiscount` column
    2. If the line item type is `SavingsPlanRecurringFee` then we take the `discounts/TotalDiscount` column
    3. For other line item types, we take which ever is available from the columns `savingsPlan/SavingsPlanEffectiveCost`, `reservation/EffectiveCost` and  `lineItem/UnblendedCost` in that order plus the `discounts/TotalDiscount` column
* Only the line items with [bill type](https://docs.aws.amazon.com/cur/latest/userguide/billing-columns.html#b-B)
`Anniversary` are taken into the calculation. In other words, line items with bill type `Purchase` and `Refund` are excluded.

### IAM User Configuration (Cost & Usage Report)

In order for meshStack to process AWS Cost and Usage Reports, following policies are required:

1. **Bucket Access Policy**: This policy allows the Metering IAM user to fetch AWS cost and usage reports
2. **Organization Access Policy**: This policy allows the Metering IAM user to list all accounts in the organization
3. **Assume Role Policy**: This policy allows Metering IAM user to assume the IAM Role `MeteringRole`

<!--DOCUSAURUS_CODE_TABS-->
<!--Bucket Access Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-cur-bucket-name",
                "arn:aws:s3:::my-cur-bucket-name/*"
            ]
        }
    ]
}
```
<!--Organization Access Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "organizations:ListAccounts",
            "Resource": "*"
        }
    ]
}
```
<!--Assume Role Policy-->
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::123456789123:role/MeteringRole"
        }
    ]
}
```
<!--END_DOCUSAURUS_CODE_TABS-->

The 3 policies in the previous section should be attached to this role.

In order for the Metering IAM user to assume this role, following trust policy is required:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789123:user/Metering"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    }
  ]
}
```

### Configuration Example

```dhall
 let example
  : AwsPlatformKrakenConfiguration
  = { platform = "aws.aws-location"
    , region = "eu-central-1"
    , meshfed-service-user =
      { access-key = Secret.Native "AWS_ACCESS_KEY_KRAKEN"
      , secret-key = Secret.Native "AWS_SECRET_KEY_KRAKEN"
      }
    , organization-root-account-role =
        "arn:aws:iam::123456789123:role/MeteringRole"
    , organization-root-account-external-id = Some
        "abcd1234-12ab-12ab-12ab-abcdef123456"
    , data-source =
        DataSource.CostAndUsageReport
          { cost-and-usage-report =
            { name = "my-cur-name"
            , bucket-name = "my-cur-bucket-name"
            , bucket-region = "eu-central-1"
            , report-key-prefix = "/my-cur-prefix"
            , cost-item-write-batch-size = 10000
            , max-report-file-processing-retries = 5
            , report-file-process-retry-delay-millis = 120000
            , max-bulk-insert-retries = 5
            , bulk-insert-retry-delay-millis = 5000
            , cost-item-read-page-size = 2000
            , apply-discounts = False
            , reported-line-item-types =
              [ "DiscountedUsage"
              , "Fee"
              , "Usage"
              , "SavingsPlanCoveredUsage"
              ]
            }
          }
    }
```
