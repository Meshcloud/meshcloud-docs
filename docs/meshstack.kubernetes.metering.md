---
id: meshstack.kubernetes.metering
title: Metering
---

meshStack supports metering and billing for most common Kubernetes versions.

## Prerequisites

- The "meshfed-metering" service account must be configured as described [here](./meshstack.kubernetes.index.md#metering)
- CPU and memory limits have to be defined for all containers, as metering is currently based on these limits
- Kubernetes APIs are accessible by meshStack metering collector components

## Supported Resources

Resources with the following traits are addressable in the [Product Catalog](meshstack.billing-configuration.md#defining-a-custom-product-catalog). Operators can use the traits of these resources to define fine-granular product and pricing rules.

### Pod

<!--snippet:mesh.kraken.productcatalog.traits.kubernetes.pod#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let PodResourceTraits =
    {-
      milliCpu:
        Configured total CPU limit of all containers in this Pod, in milli CPU (1/1000 of a CPU).

      ramMb:
        Configured total RAM limit of all containers in this Pod, in MiB.

      status:
        Status phase of this pod. See https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
    -}
      { milliCpu : Integer, ramMb : Integer, status : Text }
```
<!--Example-->
```dhall
let example
    : PodResourceTraits
    = { milliCpu = +2500, ramMb = +1024, status = "Running" }
```
<!--END_DOCUSAURUS_CODE_TABS-->
