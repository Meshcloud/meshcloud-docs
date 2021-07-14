---
id: meshstack.meshmarketplace.metering
title: Metering
---

meshStack supports metering of meshMarketplace services. This allows Service Owners to collect charges using
meshStack's multi-cloud chargeback process.

## Product Catalog Configuration

Serice Owners that want to charge consumers of their services via meshStack need to expose cost information
in the service catalog of their Service Broker implementations. meshStack metering will then automatically calculate charges
and generate a [Tenant Usage Report](./meshcloud.project-metering.md) for each marketplace used in a meshProject.

In the context of meshMarketplace metering Service Owners are also called **Sellers**. The id of a seller is the
identifier of the meshCustomer that has registed the Service Broker with the meshMarketplace.

### OSB API Service Catalog

The Service Broker's OSB API service catalog must provide cost information for each service plan as described in the [OSB API spec](https://github.com/openservicebrokerapi/servicebroker/blob/v2.15/profile.md#service-metadata). The code snippet below provides an example for describing
the cost of a service plan.

```json
{
  "plans":[
    {
      "id":"024f3452-67f8-40bc-a724-a20c4ea24b1c",
      "name":"bunny",
      "description":"A mid-sided plan.",
      "metadata":{
        "bullets":[
          "20 GB of messages",
          "20 connections"
        ],
        "costs":[
          {
            "amount":{
              "eur":99.0
            },
            "unit":"MONTHLY"
          },
          {
            "amount":{
              "usd":1000.00
            },
            "unit":"SETUP FEE"
          }
        ],
        "displayName":"Big Bunny"
      }
    }
  ]
}
```

meshStack interprets the `costs` property for each plan and automatically maintains corresponding entries in the
[meshStack metering product catalog](meshstack.billing.md#defining-a-custom-product-catalog).

> meshStack relies on Service Owners to provide accurate cost information in their service plans. When generating
> Tenant Usage Reports, meshStack metering uses the latest available cost information in the meshStack product catalog.
> Service Owners must thus be careful to ensure that any price changes are communicated to service consumers in advance
> and get published in OSB API service catalogs after usage reports of the previous period were finalized.

The following sections detail how meshStack interprets OSB API plan [cost objects](https://github.com/openservicebrokerapi/servicebroker/blob/v2.15/profile.md#cost-object) to create meshStack metering product catalog entries.

### Metrics-based Metering

See [Metrics-based Metering](meshstack.meshmarketplace.metrics-metering.md) for details about how to charge your services usage-based. This could be relevant for you if you want to charge e.g. based on actual storage or memory used by the service. Further use-cases are mentioned in the linked chapter.

> You have to make sure that the metric endpoints are available reliably. When Tenant Usage Reports are
> finalized, metrics that haven't been received until that point in time for the previous month won't be
> considered in the Usage Reports and Chargeback Statements. Finalization of Usage Reports usually happens
> four days after the end of the month. Please contact your Cloud Foundation team to find out how this offset
> is configured in your meshStack.

### Supported Unit Types

[Cost objects](https://github.com/openservicebrokerapi/servicebroker/blob/v2.15/profile.md#cost-object) in the OSB API specification support only a single `unit` field. However, service owners can specify multiple cost objects
for each plan so that different types a single service instance can incur multiple different types of charges simultaneously.

> Service Brokers must not publish multiple cost objects with the same `unit` value on any given service plan, i.e. a `unit` value can only be used once per plan.

#### Time-based Units

meshStack supports the following `unit` values for time-based metering.

```text
HOURLY
DAILY
WEEKLY
MONTHLY
YEARLY
```

meshStack metering normalizes all time units specified in OSB API service catalogs to an hourly rate using the
following conversion table:

| unit      | hours      |
| --------- | ---------- |
| `HOURLY`  | 1 h        |
| `DAILY`   | 24 h       |
| `WEEKLY`  | 24 h * 7   |
| `MONTHLY` | 30 * 24 h  |
| `YEARLY`  | 365 * 24 h |

Service instances are charged for each started hour using the normalized hourly rate from the moment a user initiates provisioning of the service instance until it is deleted.

#### Quantity-based Units

Support for charging services based on quantity units like "GB of messages stored" is forthcomming as part of our Metrics-Based Marketplace Metering feature.
Please consult meshcloud's [public roadmap](https://www.meshcloud.io/product/) for more details.

#### Setup Fees

Setup fees allow service owners to charge consumers a flat-fee for provisioning a new service instance. This is useful
if provisioning a service instance incurs a specific one-time overhead (e.g. because it's a very expensive operation or even involves manual work).

Service Owners need to use the following `unit` value to designate a setup fee:

```text
SETUP FEE
```

Note that service plans can have multiple cost objects, so it's possible to e.g. charge an initial setup fee for a
service instance while also charing a monthly usage fee.

#### Flat Fees

meshStack metering will interpret any other `unit` type value not specified above as a "flat fee".
Flat fees always incur the same charge in each reporting period, regardless of the length that the service instance
was actually used in the period. For example, a service instance that was provisioned on the last day of reporing period #1
and deleted on the first day of reporting period #2 would incur a full charge of the flat fee in each reporting period.

### Configuring "out of scope" Sellers

meshStack operators can mark sellers as "out of scope" via configuration. Once a seller is marked as out of scope,
the costs for that seller's products will be set to zero in the meshStack metering product catalog.

The line items for the seller will appear in the usage reports and chargeback statements with an appended "Out of Scope" suffix on the usage type. The total column for those line items will be zero, but the used quantity will be shown correctly.

This feature can be used for services for which usage should be tracked but the chargeback process is not yet completely established.


The following configuration options are available at `mesh.kraken.meshMarketplace`:
<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let MeshMarketplace =
    {-
        outOfScopeMarketplaceSellers:
            A list of seller ids that are "out of scope" for meshMarketplace metering.
            All usage rates for service plans published by these sellers will be replaced with 0.0.

            Note: Seller id's are the meshCustomer identifiers of the meshCustomer owning the Service Broker
            registration
    -}
      { outOfScopeMarketplaceSellers : List Text }
```
<!--Example-->
```dhall
let example
    : MeshMarketplace
    = { outOfScopeMarketplaceSellers = [ "demo-seller" ] }
```

## Reviewing Metering Data

Metering & Usage information for Serivce Owners is available for your Service Brokers from the "Marketplace Development" area in your customer Account.

You see a list of all plans  of your Service Broker's services provisioned in projects per period (usually monthly). This information is available individually per Marketplace you published your Service Broker to. Click the "Metering & Usage" Button of the according Service Broker to see the Metering & Usage data. You can filter by several criteria like period or service name.

The screen provides you with information about:

- Which customers and projects are using your services?
- How long have certain plans been used in projects?
- What are the costs per plan and project for one period?

## Seller Usage Report (CSV Export)

All the metering usage data for your services can be downloaded as CSV. This is called a seller usage report in meshcloud because the service owner acts as a seller on the meshMarketplace.

The download can be triggered directly from Metering & Usage information page. The data transformations like sorting and filtering are also included. Additional meta information can be configured beneath the metering usage information. This information will only be visible within the exported CSV document.
