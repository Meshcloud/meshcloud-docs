---
id: meshstack.configuration
title: Configuration
---

meshcloud will typically operate your meshStack installation as a [managed service](./meshstack.managed-service.md) for you. As a managed service, all configuration and validation is done by meshcloud. Nonetheless, we make references to configuration options in the documentation so that operators get a better understanding of meshStack's capabilities. The configuration references also simplify examples and communicate the exact parameters that may need to be supplied by platform operators (e.g. Service Principal credentials).

meshcloud configures your meshStack installation using a [dhall](https://dhall-lang.org/) configuration model. As part of meshcloud's managed service, customers get access to their configuration in a git repository. This is also useful to communicate configuration options and track changes.

The configuration documentation will occasionally also make references to [YAML](https://en.wikipedia.org/wiki/YAML) configuration options. These will be replaced with `dhall` models in the next releases. Dhall models can generate YAML configuration files dynamically, but provide superior features in terms of flexibility and validation.

## Global Configuration Options

### Identifiers

Objects in meshcloud can have three kinds of properties used to refer to them.

- **ID**: An immutable id, unique in space and time, e.g. `123`.
- **Identifier**: An immutable id, unique in space. Drawn from a reduced character set, e.g. `my-project`.
- **Name/Display Name**: A mutable display string to be used for display purposes e.g. `My Project`

meshStack can restrict legal identifiers for [meshCustomers](./meshcloud.customer.md) and [meshProjects](./meshcloud.project.md). This is useful to ensure projects can be identifiers are compatible with all connected cloud platforms and don't require additional name mangling to comply with cloud specific naming rules. You can configure these options in `meshfed.web.restriction` as follows:

```dhall
{ customerIdentifierLength :
    Optional Natural
, projectIdentifierLength :
    Optional Natural
, projectIdentifierPrefix :
    Optional Text
, projectNamePrefix :
    Optional Text
, envIdentifier :
    Optional Text
}
```

### Customer Registration

Multiple options are available to control how [meshCustomers](./meshcloud.customer.md) can sign up to meshStack in
self-service. meshStack can be configured to suit your organization's unique demands for sign up.

<!--snippet:mesh.panel.environment.mesh.registration-->

The following configuration options are available at `mesh.panel.environment.mesh.registration`:
<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let Registration =
    {-
        requirePayment:
            Determines if the customer registration wizard will collect a default "Cost Center" payment method.
            Disabling this will cause new meshCustomers to be registered without a payment method.
            This is useful if the meshStack implementation requires customers to register payment methods via an external
            process (e.g. via API createLimitedPaymentMethod).

            See ui.costCenter for customizing the default cost center payment method.

        externalRegistrationUrl:
            If set, disables the self-service customer registration wizard and instead redirects to the specified external
            url. Use this if the meshStack implementation uses another way to register customers (e.g. via API).
    -}
      { requirePayment : Bool
      , externalRegistrationUrl : Optional Text
      }
```
<!--Example-->
```dhall
let example =
        { requirePayment = True
        , externalRegistrationUrl = Some
            "https://itsm.example.com/order/meshcloud"
        }
      : Registration
```
<!--END_DOCUSAURUS_CODE_TABS-->

Additional configuration options control backend behavior in `meshfed.web.register` as follows:

```dhall

{ {- Configure an additional BCC email to receive registration related email notifications (e.g. a group inbox) -}
  bccEmail: Optional Text
  {- Allow sign up only if valid payment information was provided during registration  -}
, requirePayment : Optional Bool
  {- Allow generation of invite links (see below) -}
, allowPartnerInviteLinks : Optional Bool
  {- Require manual approval of new meshCustomer accounts by a partner before they can use cloud resources -}
, approvalRequired : Optional Bool
}
```

Additional remarks and configuration links:

- `requirePayment` must be consistently configured between `panel` and `meshfed` configuration settings. The configuration model validates this.
- `allowPartnerInviteLinks` enables the use of [invite links](administration.customers.md#invite-customer-via-link)
- `approvalRequired` configures manual [customer approval](./administration.customers.md#approve-customer) through a partner


### Customer User Invitations

When a user is invited to a customer there are several configurations to customize this invitation flow which is explained below.

Of importance is the setup of the new user EUID (external user-id) property. This value identifies the user against external systems during project replication. The EUID is updated for every log in of the user into meshPanel. However, to directly start replicating a newly invited user, without requiring the user to login at least once, several methods can be set up to fetch the EUID from an external identity provider.

```dhall
{ web :
  , user :
    Optional
      { rolerequest :
        {- If set to a number higher then 1, then at least that many additional
        administrator roles in a customer must approve a users role request.
        Setting it to 2 basically activates the 4-eye-principle for customer
        user role assignments. -}
        { minApprovalCount : Optional Natural

        {- If set to true the user must accept the role invitation by clicking
        on an email link, otherwise he automatically gets the role assigned. -}
        , userRoleInviteAcceptanceRequired : Optional Bool

        {- Enables to add users with E-Mails from outside the used directory.
        Depending on the cloud environment this is handled differently (in
        Azure these users are added as guest users in the AAD)  -}
        , enableGuestUser : Optional Bool

        {- If set to true the users E-Mail is assumed as EUID which allows
        GCP and Azure projects to directly replicate the users permission without
        requiring him to login into the panel first -}
        , setEmailAsEuid : Bool

        {- If set to true, all possible ways of granting the Customer Admin role via
        the panel will be prohibited. This means that the Customer Admin role can only
        be assigned via the meshObject API. This is useful when an external system
        is the source of truth regarding Customer Admin role assignments. This value
        is 'false' by default. If someone uses the 'Add Myself' button in the partner
        area, this person will be assigned as 'Customer Employee' instead. -}
        , restrictCustomerAdminRoleAssignment : Optional Bool
        }

      , revocation : Optional UserRevocation
      , identity-lookup : Optional IdentityLookup
      }
}
```

#### User Setup Steps

In order to use Azure lookup functionality, you must create a new principal (described in **Replicator** &rarr; **AAD Level Permissions** step 1 and 2) and assign the following required permissions as an **application permission**:

- `User.Read.All`

> You will also need to grant admin consent in AAD in order to activate the  `User.Read.All` permission.

If you have an Azure AAD as an upstream IDP and want to use it for user lookup you must provide meshcloud with the following credentials:

<!--snippet:mesh.meshfed.identity.azure.creds#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzureCreds =
    {-
      Setting this configuration enables the use of an AAD as a user lookup source to allow
      autocomplete of user information when adding new users to customers.

        aad-tenant:
            The active directory tenant. Its either a UID of the AAD or its domain
            like devmeshcloud.onmicrosoft.com

        client-id:
            The client id of the service principal

        client-secret:
            The credentials of the service principal

        user-lookup-strategy:
            The lookupstrategy which is used in order to find the user. Use either
            UserByUsernameLookupStrategy or UserByMailLookupStrategy. The UserByMailLookupStrategy
            uses the users euid and uses it as an E-Mail address for AAD lookup. The
            UserByUsernameLookupStrategy assumes the euid is an userPrincipalName for AAD lookup.
    -}
      { aad-tenant : Text
      , client-id : Text
      , client-secret : Secret.Type
      , user-lookup-strategy : AzureLookupStrategy
      , guestLookup : Optional AzureGuestDetection
      , euidSchemaExtensionUpdate : Optional AzureEuidExtensionSchema
      , euidUserAttributeUpdate : Optional AzureEuidUserAttribute
      , usernameAttributeUpdate : Optional AzureUsernameAttributeUpdate
      }
```
<!--Example-->
```dhall
let example
    : AzureCreds
    = { aad-tenant = "devmeshcloud.onmicrosoft.com"
      , client-id = "f112f31-248a-4461-1269-0f13164acb95"
      , client-secret = Secret.fromAnsible "client_secret"
      , user-lookup-strategy =
          AzureLookupStrategy.UserByMailLookupStrategy
      , guestLookup = None AzureGuestDetection
      , euidSchemaExtensionUpdate = None AzureEuidExtensionSchema
      , euidUserAttributeUpdate = None AzureEuidUserAttribute
      , usernameAttributeUpdate = None AzureUsernameAttributeUpdate
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

If this initial config if present you can decide to setup the following optional user identity steps:

<!--snippet:mesh.meshfed.identity.azure.guest#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzureGuestDetection =
    {-
      When adding/inviting a new user to a customer a custom attribute property from the AAD data
      can be used to determine if it is a guest user.
      Attention: This check is only performed on the first attempt when a user is added/invited
      to a customer. If this check is configured after some users were initially added to a
      customer they are not detected as guest users.

        guestProperty:
            The AAD's custom attribute which is checked against the guestValue.

        guestValue:
            If the AAD custom attribute matches this value the user is considered to be a guest.
    -}
      { guestProperty : Text, guestValue : Text }
```
<!--END_DOCUSAURUS_CODE_TABS-->

<!--snippet:mesh.meshfed.identity.azure.euid-extension-schema#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let AzureEuidExtensionSchema =
    {-
      When adding/inviting a new user to a customer a custom attribute property from the users AAD
      extension schema can be used to fill in his euid.

      Attention: This check is only performed on the first attempt when a user is added/invited
      to a customer. If this check is configured after some users were initially added to a
      customer their euid's are not udpated.

        euidExtensionSchemaProperty:
            The name of the extension schema property which is used as EUID value.
        euidExtensionSchemaIdentifier:
            The identifier of the custom schema which should be used to extract the value out of. E.g: extkvbmkofy_mySchema
            See: https://docs.microsoft.com/en-us/graph/api/schemaextension-post-schemaextensions
    -}
      { euidExtensionSchemaProperty : Text
      , euidExtensionSchemaIdentifier : Text
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

In order to use Google Cloud Directory as a lookup provider you need to provide these credentials:

<!--snippet:mesh.meshfed.identity.gcd.creds#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let GcdCreds =
    {-
      Setting this configuration enables the use of an GCD as a user lookup source to allow
      autocomplete of user information when adding new users to customers.

        domain:
            The domain used for cloud identity directory-groups created and managed by meshStack.
            meshStack maintains separate groups for each meshProject role on each managed GCP project.

        customer-id:
            The client id of the service principal

        service-account-credentials-b64:
            The credentials of the service principal
    -}
      { domain : Text
      , customer-id : Text
      , service-account-credentials-b64 : Secret.Type
      }
```
<!--Example-->
```dhall
let example
    : GcdCreds
    = { domain = "example.com"
      , customer-id = "customer-id"
      , service-account-credentials-b64 =
          Secret.fromAnsible "gcp_credentials"
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

The GCD Service User needs read access to the [GCD Directory API](https://developers.google.com/admin-sdk/directory/v1/get-start/getting-started).

If this initial config if present you can decide to setup the following optional user identity steps:

<!--snippet:mesh.meshfed.identity.gcd.euid#type-->


<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let GcdEuid =
    {-
      When adding/inviting a new user to a customer a custom attribute property from the users GCD
      custom schema can be used to fill in his euid.

      Attention: This check is only performed on the first attempt when a user is added/invited
      to a customer. If this check is configured after some users were initially added to a
      customer their euid's are not udpated anymore.

        euidCustomSchema:
            The name of the extension schema which contains the EUID value.

        euidProperty:
            The name of the custom schema property which is used as EUID value.
    -}
      { euidCustomSchema : Text, euidProperty : Text }
```
<!--Example-->
```dhall
let example
    : GcdEuid
    = { euidCustomSchema = "schema-containing-euid"
      , euidProperty = "euid-property"
      }
```
<!--END_DOCUSAURUS_CODE_TABS-->

Operators have the option of disabling inviting users that are not listed in the identity provider. They can do so by setting
the `deny-assigning-other-users` configuration option to `true`.

### Default Quotas

meshStack assigns a default quota to newly registered [meshCustomers](./meshcloud.customer.md) (see section above). Operators can configure this default quota via `meshfed.web.customer.defaultQuota`:

```dhall
{ {- the number of allowed meshProjects per meshCustomer -}
  meshProjects : Natural
}
```

The default only applies to newly registered [meshCustomers](./meshcloud.customer.md). [meshPartners](./administration.index.md) can change the individual quotas for managed meshCustomers at any time using the [administration area](administration.customers.md#customer-quota-management).

### Feedback Mailbox

Users have the option to quickly submit feedback via the meshPanel's "thumbs up/down" button in the navbar.
Operators can configure the mailbox this feedback is sent to via `meshfed.web`:

```dhall
{
  {- e.g. "feedback@example.com" -}
  feedbackEmail : Optional Text
}
```

### Message of the Day

Operators can configure an optional "message of the day" to be displayed in meshPanel.
This is useful to communicate important information such as newly available cloud platforms or known issues to every user visiting meshPanel.

Note that Platform Operators can also use [platform notifications](./administration.platforms.md#platform-notifications) as an alternative to target messages only at users that consume a specific cloud platform.

<!--snippet:mesh.panel.environment.motd-->

The following configuration options are available at `mesh.panel.environment.motd`:
<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
let MessageOfTheDay =
    {-
        message:
            The "message of the day" message you want to display. This can also include html tags.
            The "message of the day" message is shown on the home screen of meshPanel for anonymous as well
            as authenticated users.

        startTime:
            The date and time after which to begin showing the message.
            Must be a JavaScript Date.parse() compatible string.

        endTime:
            The date and time after which to stop showing the message.
            Must be a JavaScript Date.parse() compatible string.
    -}
      { message : Text, startTime : Text, endTime : Text }
```
<!--Example-->
```dhall
let example
    : Optional MessageOfTheDay
    = Some
        { message =
            "The Likvid Bank Cloud Foundation Team wishes you a meshi <a href=\"https://en.wikipedia.org/wiki/Christmas\">Christmas</a>."
        , startTime = "2019-12-23 00:00"
        , endTime = "2019-12-27 00:00"
        }

let exampleNoMessage
    : Optional MessageOfTheDay
    = None MessageOfTheDay
```
<!--END_DOCUSAURUS_CODE_TABS-->

### Customizable Payment Method Validation

As every organization has its unique accounting and cost allocation processes, it is possible to alter several aspects of the payment method functionality. For payment methods that are of the type 'COST_CENTER', the configuration below provides customization for this type of payment method (configuration is stored under `panel.ui.costCenter`)

<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
{
    {- An alias for the definition 'cost center' that your organization might use. }
    alias : Text

    {- A custom RegEx pattern that validates the user's input. }
  , validationPattern : Text

    {- The error message that should be displayed when the user's input does not match the validationPattern. }
  , validationErrorMsg : Text
}
```
<!--Example-->
```dhall
{
    alias = "ACME Center"
  , validationPattern = "\\d{6}"
  , validationErrorMsg = "You entered an invalid ACME Center number, this should be exactly six digits."
}
```
<!--END_DOCUSAURUS_CODE_TABS-->

Additionally, there is the configuration for the payment methods of type 'COST_LIMITATION'. This can be found under `panel.ui.costLimitation`

<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
{
    {- Alias(es) for the list of payment method settings that are stored as key-value pairs. This is useful for
       creating more human readable text values of certain payment method settings inside the meshPanel. }
    aliases : List { mapKey : Text, mapValue : Text }
}
```
<!--Example-->
```dhall
{
    aliases =
    [ { mapKey = "costCenter", mapValue = "ACME Cost Center" }
    , { mapKey = "anotherValue", mapValue = "Some custom value" }
    ]
}
```
<!--END_DOCUSAURUS_CODE_TABS-->


### Customizable Descriptions

Depending on your meshStack implementation, it may be helpful for you to customize error descriptions to guide
end-users towards resolving common problems in self-service. meshStack therefore provides the following configuration
options in `panel.ui`.

> Note: You can use sanitized HTML in all of these settings. This is useful to create links and apply minimal formatting.

```dhall
{
    {- Generic error message displayed when meshPanel can't connect to backend services. Configure this if users need to e.g. use a certain VPN or browser with CAs pre-installed.-}
    connectivityError : Text
    {- Displayed when users are required to sign in before registering in self-service. Configure this to explain users which IDP/Credential they need to provide -}
  , idpRegister : Optional Text

    {- Displayed when users are required to sign in before accepting an invite. Configure this to explain users which IDP/Credential they need to provide -}
  , idpAcceptCustomerUserRoleRequest : Optional Text
    {- Displayed in when users provide a cost center that's already used by another meshStack customer. Configure this e.g. to encourage users to register only one meshCustomer per cost center. -}
  , duplicateCostCenter : Optional Text
    {- Display a configurable confidentiality label in the meshPanel navbar, e.g. "internal" or "secret". -}
  , confidentialityLevel : Optional Text
    {- Display a notification that users accept platform ToS when they create a meshProject. Configure this if you need to inform end-users about custom ToS that apply to cloud platforms available in your meshStack implementation.. -}
  , platformTermsNote : Optional Text
}
```

### Email Server

For all email based communication (e.g. for [customer registration](#customer-registration) or [customer user invitations](#customer-user-invitations)) meshStack will use the configured SMTP server.

<!--snippet:mesh.meshfed.mail-->

The following configuration options are available at `mesh.meshfed.mail`:
<!--DOCUSAURUS_CODE_TABS-->
<!--Dhall Type-->
```dhall
{-
    The configured smtp mail server is used by meshfed to
    send out notifications.

    enabled:
        If True, meshfed will send out mails.

    username:
        Username meshfed uses to log in at SMTP server.

    password:
        Password meshfed uses to log in at SMTP server.

    host:
        SMTP server address that meshfed connects to.

    port:
        The port that the SMTP server listens on.

    sender:
        address:
            Address that is shown as "sender"
        reply-to:
            Address that is shown as "replyTo"

    smtp:
        ssl:
            If True, enables SSL encryption (must be supported by SMTP server)

        starttls:
            If True, enables STARTTLS encryption (must be supported by SMTP server)

        auth:
            If True, enables authentication (must be supported by SMTP server)

    theme:
        show-social:
            If True, adds configured social icons to email messages

        logo:
            href:
                Location of the logo file

            url:
                Url that the image links to

        bg-color:
            Color hex code for background

        div-color:
            Color hex code for div-containers

        messages:
            List of mappings to configure template messages

-}
  { enabled : Bool
  , username : Text
  , password : Secret
  , host : Text
  , port : Natural
  , sender : { address : Text, reply-to : Optional Text }
  , smtp :
      { ssl : { enable : Bool }, starttls : { enable : Bool }, auth : Bool }
  , theme :
      { show-social : Bool
      , logo : { href : Text, url : Text }
      , bg-color : Text
      , div-color : Text
      }
  , messages : List { mapKey : Text, mapValue : Text }
  }
```
<!--END_DOCUSAURUS_CODE_TABS-->
