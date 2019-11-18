---
id: meshstack.aws.index
title: Integration
---

*AWS* is a proprietary public cloud platform provided by Amazon Web Services. meshStack supports project and user management for AWS to include AWS services into cloud projects managed by meshStack.

meshStack supports project creation, configuration, user management and SSO for AWS.

## Integration Overview

To enable integration with AWS, Operators deploy and configure the meshStack AWS Connector. Operators can configure one or multiple `PlatformInstance`s of `PlatformType` AWS. In a typical setup, we recommend grouping each AWS `PlatformInstance` in a separate `Location`.

This makes AWS available to meshProjects like any other cloud platform in meshStack.

meshStack automatically configures AWS IAM in all managed accounts to integrate SSO with [meshStack Identity Federation](./meshstack.identity-federation.md).

## Platform Instance Configuration

### AWS Root Account

meshStack uses [AWS Organizations](https://aws.amazon.com/organizations/) to provision and manage AWS Accounts for [meshProjects](./meshcloud.project.md). To use AWS with a meshStack deployment, operators will need an AWS "root" account acting as the parent of all accounts managed by meshStack.

> Security Note: The demonstrated IAM Policies implement the minimum of configuration required to produce
> a working AWS integration using meshstack AWS Connector. This setup is based on the [default AWS Organization configuration](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html).
> We advise operators to determine the specific needs and requirements for their usage of AWS and implement more restrictive
> roles and policies.

### Identifier Configuration

meshStack operators that want to use AWS must configure their deployment to restrict identifier lengths to meet AWS requirements. The maximum allowed lengths are:

```yaml
customer_identifier_length: 16
project_identifier_length: 30
```

### meshStack IAM User

The meshStack AWS Connector uses a dedicated set of IAM credentials to work with AWS APIs on behalf of meshstack. To create these credentials, create a user in IAM with these specifications:

- User name: meshfed-service
- AWS access type: Programmatic access - with an access key

Attach the following inline policy using this json

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "StsAccessMemberAccount",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::*:role/OrganizationAccountAccessRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "${privilegedExternalId}"
                }
            }
        },
        {
            "Sid": "OrgManagementAccess1",
            "Effect": "Allow",
            "Action": [
                "organizations:DescribeOrganizationalUnit",
                "organizations:DescribeAccount",
                "organizations:ListParents",
                "organizations:ListOrganizationalUnitsForParent",
                "organizations:CreateOrganizationalUnit",
                "organizations:MoveAccount"
            ],
            "Resource": [
                "arn:aws:organizations::*:account/o-*/*",
                "arn:aws:organizations::${awsOrgAccountId}:root/o-*/r-*",
                "arn:aws:organizations::*:ou/o-*/ou-*"
            ]
        },
        {
            "Sid": "OrgManagementAccess2",
            "Effect": "Allow",
            "Action": [
                "organizations:ListRoots",
                "organizations:ListAccounts",
                "organizations:CreateAccount",
                "organizations:DescribeCreateAccountStatus"
            ],
            "Resource": "*"
        }
    ]
}
```

The `${awsOrgAccountId}` is the ID of the AWS Organization Root account under which all the created accounts are placed.

Operators should generate a unique and random value for `${privilegedExternalId}`, e.g. a GUID. meshStack AWS Connector is architected
to supply this [ExternalId](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html) only
when accessing organization member accounts from a privileged (system) context. Using the ExternalId therefore increases
the security of member accounts in your organization.

Operators need to securely inject the generated credentials and `${privilegedExternalId}` into the configuration of the AWS Connector.

### Project-Account Email Addresses

AWS requires a unique email address for each account. Operators must thus configure a wildcard email address pattern with a placeholder `%s`. The pattern must not exceed a total length of `20` characters (including the placeholder). For example, this pattern

```yaml
accountEmailTemplate: aws+%s@meshcloud.io
```

allows generation of account names:

- aws+customer.projectA@meshcloud.io
- aws+customer.projectB@meshcloud.io

### AWS Role Mapping

The [project roles](meshcloud.project.md#project-roles) are mapped to user roles in AWS. This mapping is fully customizable. It is also possible to attach a AWS policy automatically to the user role in AWS.

In order to configure the mapping use the following configuration:

#### Aws.dhall

```dhall
in  P.Platform.Aws
  { platform = "aws.test-aws"
  , region = "us-east-1"
  , roleMappings = [
    { mapKey = "admin" // Name of project role in meshStack
    , mapValue =
      { roleName = "CustomRole" // AWS Role Name
      , policyArn = None Text // Optional: The policy which is attached to the role.
      }
    } ]
```

If the role does not exist in AWS the replicator tries to create it. It also is setting up a trust relationship to the IdP in order to allow SSO for the project users.

If the AWS role does already exist the replicator will update the IdP trust relationship and (if configured) attach the policy via its ARN. Already attached policies to the role won't be changed.

### Account Alias

Accounts in AWS get an alias assigned. This alias is fully customizable. You can use the placeholder `<customer>` and `<project>`, they are replaced with the customer and the project identifier during replication.



 In order to do so set the following config:

| Aws.dhall             | Description                                                                                          |
| --------------------- | :--------------------------------------------------------------------------------------------------- |
| `accountAliasPattern` | Controls the name of the AWS Account alias. You can use the placeholder `<customer>` and `<project>` |
