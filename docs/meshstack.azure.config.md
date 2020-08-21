---
id: meshstack.azure.config
title: Configuration Reference
---

This section describes the configuration of an Azure Platform Instance in the meshStack configuration model at `mesh.platforms`.

For easier reference the following sections break down the configuration model in multiple parts. The union of these defines the full configuration model.

<!--snippet:mesh.platform.azure#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzurePlatformConfiguration =
        AzurePlatformCoreConfiguration
      ⩓ AzurePlatformBlueprintConfiguration
      ⩓ { servicePrincipal : ServicePrincipal
        , b2bUserInvitation : Optional InviteB2BUserConfig
        , provisioning : Provisioning
        , roleMappings : List RoleMapping
        }
```
<!--END_DOCUSAURUS_CODE_TABS-->

## Core Configuration

<!--snippet:mesh.platforms.azure.core#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzurePlatformCoreConfiguration =
    {-
      platform:
        The meshPlatform identifier

      subscriptionNamePattern:
        Configures the pattern that defines the desired name of Azure Subscriptions managed by meshStack.
        This Java String.format format string receiving the following arguments:

          1. meshCustomer identifier
          2. meshProject identifier
          3. meshProject ID (numeric)
          4. tenantNumber (numeric), a running number specific to each platform which can be optionally enabled

        Operators must ensure the resulting subscription names are unique in the managed AAD Tenant.

      groupNamePattern:
        Configures the pattern that defines the desired name of AAD groups managed by meshStack.
        This Java String.format format string receiving the following arguments:

          1. meshCustomer identifier
          2. meshProject identifier
          3. meshProject ID (numeric)
          4. tenantNumber (numeric), a running number specific to each platform which can be optionally enabled
          5. role name suffix (configurable via Landing Zone)

        Operators must ensure the group names are unique in the managed AAD Tenant.
    -}
      { platform : Text
      , subscriptionNamePattern : Optional Text
      , groupNamePattern : Optional Text
      }
```
<!--Example-->
```dhall
let example
    : AzurePlatformCoreConfiguration
    =
      -- creates subscription names like "customer.project"
      -- and group names like "customer.project-reader"
      { platform = "azure.mylocation"
      , subscriptionNamePattern = Some "%s.%s"
      , groupNamePattern = Some ".%s.%s-%5\$s"
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->


## Blueprint Configuration

<!--snippet:mesh.platforms.azure.blueprint#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzurePlatformBlueprintConfiguration =
    {-
        blueprintServicePrincipal:
          Object Id of the Enterprise Application belonging to the Microsoft Application "Azure Blueprints" with Application Id
          f71766dc-90d9-4b7d-bd9d-4499c4331c3f in the managed AAD Tenant. meshStack will grant the necessary permissions on
          managed subscriptions to this SPN so that it can create System Assigned Managed Identities (SAMI) for Blueprint execution.

          Note: Operators can also explicitly configure an UAMI to be used instead of a SAMI in Azure Landing Zone definitions.
          If an UAMI is not explicitly configured, meshStack will default to using the SAMI.

        blueprintLocation:
          The Azure location where replication creates and updates Blueprint Assignments. Note that it's still
          possible that the Blueprint creates resources in other locations, this is merely the location where the
          Blueprint Assignment is managed.
    -}
      { blueprintServicePrincipal : Text, blueprintLocation : Text }
```
<!--Example-->
```dhall
let example
    : AzurePlatformBlueprintConfiguration
    = { blueprintServicePrincipal = "2a6a62ad-e28b-4eb4-8f1e-ce93dbc76d20"
      , blueprintLocation = "westeurope"
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

## Service Principal Configuration

This section configures the Service Principal (SPN) used by meshStack's replicator
to automate project replication. The SPN needs to be configured in the same
AAD Tenant that is used to hold all subscriptions managed by this meshPlatform.

<!--snippet:mesh.platform.azure.servicePrincipal#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let ServicePrincipal =
    {-
      Configures an Azure Service Principal.

          aadTenant:
              Domain name or Id of the AAD Tenant that holds the Service Principal.

          objectId:
              The Object Id of the Service Principal. In Azure Portal, this is the Object Id
              of the "Enterprise Application".

          clientId:
              The Application Client Id. In Azure Portal, this is the Application Id of the
              "Enterprise Application" but can also be retrieved via the "App Registration" object
              as "Application (Client) Id".

          clientSecret:
              A valid secret for accessing the SPN. In Azure Portal, this can be configured on the
              "App Registration" objecct.
    -}
      { aadTenant : Text
      , objectId : Text
      , clientId : Text
      , clientSecret : Secret
      }
```
<!--Example-->
```dhall
let example =
      { aadTenant = "example.onmicrosoft.com"
      , objectId = "655985db-ca43-4b9f-b317-9dd3d0289c50"
      , clientId = "2a51f406-27fd-4e40-8bd3-fe83ff934a47"
      , clientSecret = Secret.Native "AZURE_CLIENT_SECRET"
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

### B2B User Invitation

You can decide if you want Azure to send an automatic email notification about the invitation process to the user by setting `send-azure-invitation-mail` to `true`. Usually this is not needed as meshStack handles
the invitation notifications.

This configuration is optional. When configured, instructs the replicator to create AAD B2B guest invitations for
users missing in the AAD tenant managed by this meshPlatform.

> B2B Invitations require an email address which is usually fetched from the [euid](./meshstack.identity-federation.md#externally-provisioned-identities). 

<!--snippet:mesh.platform.azure.inviteB2BUserConfig#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let InviteB2BUserConfig =
    {-
      redirectUrl:
          URL used in the Azure invitation mail if send

      sendAzureInvitationMail
          Flag if an Invitation mail by Azure should be send out
    -}
      { redirectUrl : Text, sendAzureInvitationMail : Bool }
```
<!--Example-->
```dhall
let example
    : Optional InviteB2BUserConfig
    = Some
        { redirectUrl = "https://portal.azure.com/#home"
        , sendAzureInvitationMail = True
        }

let exampleNoB2bInvites
    : Optional InviteB2BUserConfig
    = None InviteB2BUserConfig
```
<!--END_DOCUSAURUS_CODE_TABS-->

## Provisioning Configuration

meshStack allows operators to control how the replicator provisions subscriptions for meshProjects. meshStack supports automated provisioning from an existing Enterprise Agreement (EA) Enrollment Account or consuming pre-provisioned subscriptions created from other sources.

Pre-provisioned subscriptions are useful when no EA is available and operators want to consume CSP or Pay-as-you-go Subscriptions instead.

<!--snippet:mesh.platform.azure.provisioning#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let SubscriptionOwner =
    {-
      This configuration is available on all provisioning types.

          subscriptionOwnerObjectIds:
              One or more principals Object Ids (e.g. Groups, SPNs) that meshStack will ensure have an "Owner" role
              assignment on managed subscriptions. This can be useful to satisfy Azure's constraint of at least
              one direct "Owner" role assignment per Subscription.
    -}
      { subscriptionOwnerObjectIds : List Text }

let EnterpriseEnrollment =
    -- Creating new Subscriptions under an Enterprise Agreement (EA) Enrollment Account.

      let EnterpriseEnrollmentConfig =
          {-
            enrollmentAccountId:
                Object Id of the EA Enrollment Account.

            subscriptionOfferType:
                The Microsoft Subscription offer type to use when creating subscriptions.
                Common values are "MS-AZR-0017P" for standard and "MS-AZR-0148P" for Dev/Test subscriptions.

            subscriptionCreationErrorCooldownSec:
                There is a safety mechanism to avoid duplicate Subscription creation in case
                of an error on Azure's EA API. This delay should be a bit higher then it usually takes to
                create subscriptions. For big installations this is somewhere between 5-15 minutes.
          -}
            { enrollmentAccountId : Text
            , subscriptionOfferType : Text
            , subscriptionCreationErrorCooldownSec : Natural
            }

      in    SubscriptionOwner
          ⩓ { enterpriseEnrollment : EnterpriseEnrollmentConfig }

let PreProvisioned =
    -- Look up pre-provisioned Subscriptions in the AAD Tenant based on a prefix name.

      let PreProvisionedConfig =
          {-
            unusedSubscriptionNamePrefix:
                The prefix that identifies unused subscriptions. Subscriptions will be renamed during meshStack's
                project replication, at which point they should no longer carry this prefix.
          -}
            { unusedSubscriptionNamePrefix : Text }

      in  SubscriptionOwner ⩓ { preProvisioned : PreProvisionedConfig }

let Provisioning =
      < EnterpriseEnrollment : EnterpriseEnrollment
      | PreProvisioned : PreProvisioned
      >
```
<!--Example-->
```dhall
let exampleEnterpriseEnrollment =
      Provisioning.EnterpriseEnrollment
        { subscriptionOwnerObjectIds =
          [ "390eaa30-67c3-4314-9368-33c76472e6f1" ]
        , enterpriseEnrollment =
          { enrollmentAccountId = "7eeec94c-96a5-4faa-a6d6-8999bbbfa194"
          , subscriptionOfferType = "MS-AZR-0017P"
          , subscriptionCreationErrorCooldownSec = 900
          }
        }

let examplePreProvisioned =
      Provisioning.PreProvisioned
        { subscriptionOwnerObjectIds = [] : List Text
        , preProvisioned.unusedSubscriptionNamePrefix = "unused-"
        }
```
<!--END_DOCUSAURUS_CODE_TABS-->

## Role Mappings

Role mappings instruct the replicator how to map meshProject roles to Azure RBAC role assignments
during replication. Operators can override these default role mappings in Landing Zones.

Role mappings must be configured for all [meshProject roles](./meshcloud.project.md#project-roles), 
you can refer to the official [Azure documentation](https://docs.microsoft.com/bs-latn-ba/azure/role-based-access-control/built-in-roles) for additional information on Azure roles.

<!--snippet:mesh.platform.azure.roleMapping#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let RoleMapping =
    {-
      Defines the mapping of a meshProject role to Azure RBAC Roles.

          mapKey:
              The id of the meshProject role defined in this mapping.

          mapValue.id:
              The id of an Azure RBAC Role. This can be a built-in or custom RBAC role.

          mapValue.alias:
              A custom alias for this role. The alias is used for naming IAM groups mapped to this role.
              See groupNamePattern configuration.
    -}
      { mapKey : Text, mapValue : { alias : Text, id : Text } }
```
<!--Example-->
```dhall
let example
    : RoleMapping
    =
      -- This mapping gives all users with meshProject Role "admin" an Azure "Owner" role assignment
      -- on the subscription managed for this meshProject.
      { mapKey = "admin"
      , mapValue =
        { alias = "owner", id = "8e3af657-a8ff-443c-a75c-2fe8c4bcb635" }
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->