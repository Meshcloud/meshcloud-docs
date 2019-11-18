---
id: meshstack.openshift.index
title: Integration
---

meshStack supports management of RedHat OpenShift platforms. OpenShift has a [Kubernetes](meshstack.kubernetes.index.md) core and provides additional services. It is available in both Open Source flavors (OKD) as well as enterprise offerings by RedHat.

meshStack supports project creation, configuration, user management and SSO for OpenShift.

## Integration Overview

To enable integration with OpenShift, operators deploy and configure the meshStack OpenShift Connector. Operators can configure one or multiple `PlatformInstance`s of `PlatformType` OpenShift. This makes OpenShift available to meshProjects like any other cloud platform in meshStack.

meshStack automatically configures OpenShift Projects and Permissions to integrate SSO with [meshStack Identity Federation](./meshstack.identity-federation.md).

## Prerequisites

### OpenShift Versions

meshStack currently supports OpenShift version 3.7 as either Open-Source (OKD) or OpenShift Enterprise variants.

### IdP Configuration

In order to integrate with [meshStack Identity Federation](./meshstack.identity-federation.md), operators need to configure the meshStack Identity Broker as an [OpenID Identity Provider in OpenShift](https://docs.okd.io/latest/install_config/configuring_authentication.html#OpenID) using the following settings:

```yml
identityProviders:
  - name: meshfed_oidc
    challenge: false
    login: true
    mappingMethod: claim
    provider:
      apiVersion: v1
      kind: OpenIDIdentityProvider
      clientID: meshfed-oidc
      clientSecret: ignored
      extraScopes:
      - email
      - profile
      - skipIdentifierSelection
      extraAuthorizeParameters:
        include_granted_scopes: "true"
      claims:
        id:
        - sub
        preferredUsername:
        - sub
        name:
        - preferred_username
        email:
        - email
      urls:
        authorize: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/auth?response_mode=query
        token: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/token
        userInfo: https://sso.example.meshcloud.io/auth/realms/meshfed/protocol/openid-connect/userinfo
```

### meshStack Service Accounts

The meshStack OpenShift Modules use dedicated OpenShift ServiceAccounts to work with OpenShift APIs on behalf of meshStack.
To create these credentials, create the following objects via `oc create -f <file>` as a Cluster Administrator.

> You can also use `oc replace -f <file>` to update existing definitions.

The meshStack ServiceAccounts can be located in a dedicated namespace. In the following yaml files we use the `meshcloud` namespace for the ServiceAccounts.
You can also define a different namespace if you prefer.
Before applying the yaml file, the namespace has to be created first via `oc create namespace meshcloud`.

#### Tenant Management

The tenant management component of meshStack requires the following ServiceAccount.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: meshfed-service
  namespace: meshcloud
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: meshfed-service
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
  - update
- apiGroups:
  - ""
  - user.openshift.io
  resources:
  - groups
  - identities
  - useridentitymappings
  - users
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  - project.openshift.io
  resources:
  - projectrequests
  - projects
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - resourcequotas
  - resourcequotas/status
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  - template.openshift.io
  resources:
  - templates
  - templateinstances
  - templateconfigs
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  - quota.openshift.io
  resources:
  - appliedclusterresourcequotas
  - clusterresourcequotas
  - clusterresourcequotas/status
  verbs:
  - create
  - delete
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - roles
  - rolebindings
  - clusterroles
  - clusterrolebindings
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - rolebindings
  verbs:
  - create
  - delete
  - update
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - clusterroles
  verbs:
  - bind
  resourceNames:
  - admin
  - edit
  - view
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: meshfed-service
  annotations:
    io.meshcloud/meshstack.replicator-openshift.version: "1.0"
subjects:
- kind: ServiceAccount
  name: meshfed-service
  namespace: meshcloud
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: meshfed-service
```

### Custom meshProject Roles

If you want to use custom roles to be mapped to your meshProject roles (and not just the pre-defined `admin`, `edit` and `view` roles) you need to make sure to also list these roles in the clusterrole binding section for the meshfed-service principle. It is not allowed for the service-principle to bind roles granting more rights then itself has, so the right to bind these roles must be explicitly given.

For example if you plan to use a role named `my-custom-role` please change the relevant section in the above document to:

```yml
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  - authorization.openshift.io
  resources:
  - clusterroles
  verbs:
  - bind
  resourceNames:
  - admin
  - edit
  - view
  - my-custom-role
```
