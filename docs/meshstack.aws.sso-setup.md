---
id: meshstack.aws.sso-setup
title: SSO Setup
---
## Configure AAD as IdP for AWS SSO

This section describes how to set up AWS SSO with Azure Active Directory.

### Create an Enterprise Registration

Search for `aws` on Azure and choose `AWS Single Sign-on` as the enterprise registration type

![assets/aws_sso_setup/azure_create_enterprise_registration.png](assets/aws_sso_setup/azure_create_enterprise_registration.png)

Follow the steps in AAD under Getting Started (Pick SAML for the single sign on)

![assets/aws_sso_setup/aad_getting_started.png](assets/aws_sso_setup/aad_getting_started.png)

### Configure AWS for AAD integration

Switch to AWS Console and search for `sso`, now choose `AWS Single Sign-on`

![assets/aws_sso_setup/aws_single_sign_on.png](assets/aws_sso_setup/aws_single_sign_on.png)

Now choose `Enable AWS SSO`

![assets/aws_sso_setup/aws_enable_sso.png](assets/aws_sso_setup/aws_enable_sso.png)

Configure the identity source by clicking on `Choose your identity source`

![assets/aws_sso_setup/aws_choose_identity_source.png](assets/aws_sso_setup/aws_choose_identity_source.png)

Now choose `change` on `Identity source`

![assets/aws_sso_setup/aws_change_identity_source.png](assets/aws_sso_setup/aws_change_identity_source.png)

Select `External identity provider`

![assets/aws_sso_setup/aws_set_identity_source_external.png](assets/aws_sso_setup/aws_set_identity_source_external.png)

### Metadata

Download `AWS SSO SAML metadata` from AWS and then you should upload it into `Single Sign-On` page on Azure with the `Upload metadata file` button.

![assets/aws_sso_setup/azure_upload_metadata.png](assets/aws_sso_setup/azure_upload_metadata.png)

Also check the following image to download `IdP SAML metadata` file from Azure, you can find it in the same page as `Federation Metadata XML`

![assets/aws_sso_setup/azure_download_metadata.png](assets/aws_sso_setup/azure_download_metadata.png)

After downloading the `Federation Metadata XML` from Azure you have to upload this file into your AWS SSO Identity Source under `IdP SAML metadata`

### Provisioning

Before the automatic provisioning is enabled, you can click `enable automatic provisioning` on Azure to get the Tenant URL and Access Token to configure SCIM for provisioning on AAD side.

![assets/aws_sso_setup/azure_tenant_url_secret_token.png](assets/aws_sso_setup/azure_tenant_url_secret_token.png)

The `userName` must be set to what is configured as an `euid` in meshStack

![assets/aws_sso_setup/azure_mappings.png](assets/aws_sso_setup/azure_mappings.png)

Disable the Group sync here, as usually only groups managed by meshStack will be used and they will be created by meshStack anyway.

When clicking on the directory users mapping, you will see, that the userPrincipalName to userName mapping cannot be deleted.

![assets/aws_sso_setup/azure_username_mapping.png](assets/aws_sso_setup/azure_username_mapping.png)



![assets/aad_as_idp_for_aws/aad_as_idp_for_aws_12.png](assets/aws_sso_setup/azure_mange_claim.png)

These users will be replicated with Provisioning Settings. You should also configure that so you can see your Users in your AWS SSO directory.

![assets/aad_as_idp_for_aws/aad_as_idp_for_aws_13.png](assets/aws_sso_setup/azure_replication.png)

Once you configure that Provisioning, you should click that `Start provisioning` button and it will replicate your Users and Groups into the AWS SSO. After this point this service will replicate the users every 40 minutes however you can always use `Provision on demand` or `Restart provisioning` buttons to run that replication.

![assets/aad_as_idp_for_aws/aad_as_idp_for_aws_14.png](assets/aws_sso_setup/aad_as_idp_for_aws_14.png)
