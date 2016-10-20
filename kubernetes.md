# Kubernetes Identity Management

## Introduction and Background

This page gives an in-depth analysis of Kubernetes' various authentication methods and
provides several reference architectures for deploying an identity management solution
with for Kubernetes.  We will illustrate the examples using OpenUnison, but these concepts
can be applied generically with other products and projects as well.

## SSO / Authentication

### Certificate

Certificate authentication uses TLS mutual authentication, where a client and server
both have a private key and certificate when interacting.  In many ways this mechanism
requires the least amount of infrastructure but also has several drawbacks.

*ADD IMAGE*

When authenticating, kubectl will present your certificate to the API server as your identity.  
Prior to Kubernetes 1.3 this was the most powerful authentication mechanism as it only required
a Certificate Authority to be setup.  Adding new users means just generating a new certificate
signing request and getting it signed by your CA with no changes to the Kubernetes configuration.

The major drawbacks to this approach are:

1.  No revocation - Until Kubernetes 1.5 certificate revocation is not possible, so there's no way to make sure a user can be disabled
2.  Shared certificates - Often certificates are shared across a team, creating a security risk
3.  No authorizations - As of 1.3 there is no way for a certificate to specify a set of groups, several models are proposed but none have been fully implemented
4.  No multi-factor support - There's no way to implement a multi-factor scheme with certificate based authentication, so if you have that as a requirement you may run into issues.

Where certificate authentication IS a good idea is as a "break glass in case of emergency" user.  If using OpenID Connect or implementing your own authentication keeping a certificate in case the external authentication system goes down is a good idea as an emergency fallback.

### OpenID Connect

OpenID Connect (http://openid.net/connect/) is a standard for authentication that is built on top of
OAuth 2.0, JWT and other standards.  This guide isn't going to be a definitive guide for OpenID Connect
but will help you get started.  Some important definitions:

| Term | Definition |
| ----- | --------- |
| Identity Provider (IdP) | Source of identity data |
| Relying Party (RP) | Something that will consume, or relies, on identity data |
| acccess_token | A token that is recognized by an IdP to authorize services |
| JSON Web Token (JWT) | A way to encode identity data in JSON that is digitally signed |
| Claim | An attribute in a JWT |
| id_token | A JWT containing data about the user |
| Bearer token | A token that is opaque to the user |

What makes OpenID Connect integration unique for Kubernetes is that Kubernetes does NOT authenticate you.  
This seems counter-intuitive (and counter productive) but its a very smart design.  

![Kubernetes OpenID Connect Flow](https://www.gliffy.com/go/publish/image/11149637/L.png)

1.  Login to your IdP (more on this later)
2.  Your identity provider will provide you with an access_token, id_token and a refresh_token
3.  When using kubectl, use your id_token with the --token flag
4.  The api server will make sure the JWT signature is valid by checking against the certificate named in the configuration
5.  Check to make sure the id_token hasn't expired
6.  Make sure the user is authorized

Since all of the data needed to validate who you are is in the id_token, Kubernetes doesn't need to
"phone home" to the IdP.  In a model where every request is stateless this provides a very scalable
model.  It does offer a few challenges though:

1.  Kubernetes has no "web interface" to trigger the authentication process.  There is no browser or interface to collect credentials.
2.  The id_token can't be revoked, its like a certificate so it should be short-lived (only a few minutes) so it can be very annoying to have to get a new token every few minutes
3.  There's no easy way to authenticate to the Kubernetes dashboard without using the kubectl -proxy command

We'll talk about how to address these issues in the reference architectures section.

#### Deploying an Identity Provider

There are multiple IdPs that will work with Kubernetes including (this is NOT an exhaustive list):

1.  OpenUnison (Tremolo Security)
2.  Dex (CoreOS)
3.  Keycload (Red Hat)
4.  Google
5.  AzureAD

In order for an IdP to work it must:

1.  Run in TLS
2.  Have a CA signed certificate (even if the CA is not a commercial CA)
3.  Support OpenID Connect Discovery (https://openid.net/specs/openid-connect-discovery-1_0.html)

When generating an id_token the username claim and optionally a groups claim.  For 1.3 and 1.4 your
group claim MUST be an array, even if it only has a single value.  1.5 will eliminate this requirement.

A note about requirement #2 above, requiring a CA signed certificate.  If you deploy your own IdP (as apposed to one of the cloud providers like Goolge or Microsoft) you MUST have your IdP's web server certificate signed by a CA.  This is an issue with GoLang's
TLS client implementation not being able to verify a self-signed certificate.  If you don't have a CA handy, you can use this script
from the CoreOS team to create a simple CA and a signed certificate and key pair - https://github.com/coreos/dex/blob/1ee5920c54f5926d6468d2607c728b71cfe98092/examples/k8s/gencert.sh

#### Configuring Kubernetes

Configuring Kubernetes for OIDC requires adding several parameters to the api server.  If you are looking to test our OIDC
without deploying a cluster manually, look at CoreOS' single node Vagrant image - https://coreos.com/kubernetes/docs/latest/kubernetes-on-vagrant-single.html.  Currently, minikube and other most other simple distributions do not support changing the api server parameters.

The below table details the parameters:

| Parameter | Description | Example |
| --------- | ----------- | ------- |
| --oidc-issuer-url | The base URL for the issuer.  This URL should point to the level below .well-known/openid-configuration | If the discovery URL is https://mlb.tremolo.lan:8043/auth/idp/oidc/.well-known/openid-configuration the value should be https://mlb.tremolo.lan:8043/auth/idp/oidc |
| --oidc-client-id | The name of your client as identified by your IdP | kubernetes |
| --oidc-username-claim | The name of the claim in the JWT that stores the user's ID | sub |
| --oidc-groups-claim | The name of the claim in the JWT that stores the user's group memberships | user_roles |
| --oidc-ca-file | The path to the certificate for the CA that signed your IdP's web certificate | /etc/kubernetes/ssl/kc-ca.pem |

If using the above example configuration and if your id_token was:

```
{
  "iss": "https://mlb.tremolo.lan:8043/auth/idp/oidc",
  "aud": "kubernetes",
  "exp": 1474596669,
  "jti": "6D535z1PJE62NGt1ierboQ",
  "iat": 1474596369,
  "nbf": 1474596249,
  "sub": "mwindu",
  "user_role": [
    "users",
    "new-namespace-viewer"
  ],
  "email": "mwindu@nomorejedi.com"
}
```

Then Kubernetes would identify you as mwindu with the groups "users" and "new-namespace-viewer".  **NOTE** that this is the decoded JWT.
The site https://jwt.io can be used to decode a JWT.  The encoded JWT is:

```
eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL21sYi50cmVtb2xvLmxhbjo4MDQzL2F1dGgvaWRwL29pZGMiLCJhdWQiOiJrdWJlcm5ldGVzIiwiZXhwIjoxNDc0NTk2NjY5LCJqdGkiOiI2RDUzNXoxUEpFNjJOR3QxaWVyYm9RIiwiaWF0IjoxNDc0NTk2MzY5LCJuYmYiOjE0NzQ1OTYyNDksInN1YiI6Im13aW5kdSIsInVzZXJfcm9sZSI6WyJ1c2VycyIsIm5ldy1uYW1lc3BhY2Utdmlld2VyIl0sImVtYWlsIjoibXdpbmR1QG5vbW9yZWplZGkuY29tIn0.f2As579n9VNoaKzoF-dOQGmXkFKf1FMyNV0-va_B63jn-_n9LGSCca_6IVMP8pO-Zb4KvRqGyTP0r3HkHxYy5c81AnIh8ijarruczl-TK_yF5akjSTHFZD-0gRzlevBDiH8Q79NAr-ky0P4iIXS8lY9Vnjch5MF74Zx0c3alKJHJUnnpjIACByfF2SCaYzbWFMUNat-K1PaUk5-ujMBG7yYnr95xD-63n8CO8teGUAAEMx6zRjzfhnhbzX-ajwZLGwGUBT4WqjMs70-6a7_8gZmLZb2az1cZynkFRj2BaCkVT3A2RrjeEwZEtGXlMqKJ1_I2ulrOVsYx01_yD35-rw
```

#### Kubectl

The most "standard" way to use OIDC with kubectl is to retrieve your id_token and embed it in your kubectl command:

```
$ kubectl --token=eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL21sYi50cmVtb2xvLmxhbjo4MDQzL2F1dGgvaWRwL29pZGMiLCJhdWQiOiJrdWJlcm5ldGVzIiwiZXhwIjoxNDc0NTk2NjY5LCJqdGkiOiI2RDUzNXoxUEpFNjJOR3QxaWVyYm9RIiwiaWF0IjoxNDc0NTk2MzY5LCJuYmYiOjE0NzQ1OTYyNDksInN1YiI6Im13aW5kdSIsInVzZXJfcm9sZSI6WyJ1c2VycyIsIm5ldy1uYW1lc3BhY2Utdmlld2VyIl0sImVtYWlsIjoibXdpbmR1QG5vbW9yZWplZGkuY29tIn0.f2As579n9VNoaKzoF-dOQGmXkFKf1FMyNV0-va_B63jn-_n9LGSCca_6IVMP8pO-Zb4KvRqGyTP0r3HkHxYy5c81AnIh8ijarruczl-TK_yF5akjSTHFZD-0gRzlevBDiH8Q79NAr-ky0P4iIXS8lY9Vnjch5MF74Zx0c3alKJHJUnnpjIACByfF2SCaYzbWFMUNat-K1PaUk5-ujMBG7yYnr95xD-63n8CO8teGUAAEMx6zRjzfhnhbzX-ajwZLGwGUBT4WqjMs70-6a7_8gZmLZb2az1cZynkFRj2BaCkVT3A2RrjeEwZEtGXlMqKJ1_I2ulrOVsYx01_yD35-rw get nodes
```

If you're using Google as an IdP, @micahhausler has written a simple tool - https://github.com/micahhausler/k8s-oidc-helper

### Tokens

## RBAC

RBAC, or Role Based Access Control, is an authorization model for Kubernetes that lets you define a set of permissions based on a role.  Roles can be made up of:

1.  Users
2.  Groups
3.  Service Accounts

In general, its not a good practice to add users directly to roles.  Roles are static (for 1.3 and 1.4) so adding subjects means patching or delete/create.  Also tracking who has access to what can be much more difficult (more on this later).

Roles are scoped to a specific namespace, where as a ClusterRole applies to an entire cluster.  The relationships between objects are:

| Object | Description |
| ------ | ----------- |
| Role or ClusterRole | Defines a set of urls and actions on those urls (name a namespace for Role) |
| RoleBinding or ClusterRoleBinding | Defines the subjects |
| Group / User / ServiceAccount | Data that identifies a user |

### Configure Kubernetes

To enable RBAC add the following api-server flags:

| API Flag | Description |
| -------- | ----------- |
| --authorization-mode=RBAC | Tells the API server to use RBAC for authoriztions |
| --runtime-config=extensions/v1beta1/networkpolicies=true,rbac.authorization.k8s.io/v1alpha1 | Enable the alpha API |
| --authorization-rbac-super-user=kube-admin | Identify a super-user |

The first flag tells the api server to use RBAC for authorizations.  The second flag tells the API server to enable the RBAC apis. Finally, the last flag tells the api server who the super user is.  

The only flag that requires real discussion is the super user flag.  You can not create a policy that grants you more permisions then you have.  When starting up, the api server has no policies until you create them (this will change in 1.5 with bootstrap actions).  So if users have no authorizations they can't be authorized to create policies.  This is why you need a super user.  You CAN use an OIDC user as your super user, but our recommendation is to use a certificate user for a few reasons:

1.  Certificate user's aren't reliant upon an external service
2.  A certificate user can be locked down and stored in a secure environment as a "break glass in case of emergency" user

If you do use an OIDC user, you will need to prepend the issuer to your username.  For instance if you want your super user to be mmosley from OIDC you would specify "https://mlb.tremolo.lan:8043/auth/idp/oidc#mmosley" with one exception, if your claim is an email address.

### Policy Design

Policy design is a very advanced topic and will vary greatly per deployment.  In general, keep it simple.  Very complex and granular entitlement solutions are extremely hard to manage and usually cause more harm then good.  If you find that you are creating more then three to five roles for any given deployment chances are they are too granular.

#### Initial Policy

In order to let anyone do the most basic tasks, access to certain apis is required.  The below policy api is a good start:

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: discovery
rules:
  - apiGroups: []
    resources: []
    verbs: ["get"]
    nonResourceURLs: ["/version","/api", "/api/*","/apis", "/apis/*","/apis/apps/v1alpha1","/apis/autoscaling/v1","/apis/batch/v1","/apis/batch/v2alpha1","/apis/extensions/v1beta1","/apis/policy/v1alpha1","/apis/rbac.authorization.k8s.io/v1alpha1"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: discovery-binding
subjects:
- kind: Group
  name: users
roleRef:
  kind: ClusterRole
  name: discovery
```

The first part of the YAML defines a cluster role, which is global to the entire cluster.  The nonResourceURLs were determined through trial-and-error so you may find that you need more.  NOTE: in 1.3 wildcard URLs do not work.  1.4 up will simplify this policy by eliminating specific URLs.

The second part defines who has access to the role.  We listed the group "users" to make sure that all users in the group "users" can do this.  This means that every user that logs in MUST have the group "users".

If you save this file as cluster-discovery.yaml you can deploy it:

```
$ kubectl create -f /path/to/cluster-discovery.yaml
```

#### Namespace Policies

A common pattern is to create policies that isolate namespaces.  This pattern lets you define access to an individual namespace and equate a namespace with a team or project.  This is how OpenShift isolates individual projects.  The first step would be to create a namespace:

```
$ kubectl create namespace new-namespace
```

Once the namespace is created, create your policies.  The below policies creates two roles:

1.  admin - Full access to create, run, destroy pods/routes/etc
2.  viewer - View resources only

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: admin-role
  namespace: new-namespace
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
  nonResourceURLs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: admin-binding
  namespace: new-namespace
subjects:
- kind: Group
  name: new-namespace-admin
roleRef:
  kind: Role
  name: admin-role
  namespace: new-namespace
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: viewer-role
  namespace: new-namespace
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get","view","list","watch"]
  nonResourceURLs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: viewer-binding
  namespace: new-namespace
subjects:
- kind: Group
  name: new-namespace-viewer
roleRef:
  kind: Role
  name: viewer-role
  namespace: new-namespace
```

The first block defines an admin role, with a role binding for a group named "new-namespace-admin", so a user that authenticates with a group "new-namespace-admin" will have full admin access to the new-namespace.  The second role only allows for "read" access if you have the group "new-namespace-viewer".  

To update these roles for other namespaces, just change the "namespace" label whenever you find it to the name of your namespace.

## Reference Architectures

When designing an identity management solution for Kubernetes there are several points to take into account.  We'll cover each of these topics in detail.

### How will uses authenticate?

When identifying how users will authenticate remember that Kubernetes keep the following in mind:

1.  Kubernetes has no way of triggering an authentication process
2.  Once authenticated, the id_token needs to be provided to the api-server / kubectl
3.  The id_token is short lived, so it will need to be refreshed often

In addition, identify what policies are in place such as:

1.  Does Kubernetes qualify as requiring multi-factor authentication?
2.  Do you need to follow a privileged access policy?


### How will users use kubectl?

Once you have generated an id_token, how will you use it with kubectl?  In OIDC, both the id_token and access_token are meant to be short lived.  This makes sense, as an id_token stands on its own until it expires.  If I login with an id_token that says I have admin access and the token is valid for 5 minutes then for that 5 minutes I have all the access I could want.  What if in that time I lose my admin access?  How would Kubernetes stop me from performing admin actions?  It can't, the api-server never talks to the identity provider to validate that the id_token is still ok.

#### Option 1 - OIDC Authenticator

The first option is to use a new feature in 1.4 that isn't well documented called a custom authenticator.  In this case, the oidc authenticator.  This authenticator takes your id_token, refresh_token and your OIDC client_secret and will refresh your token automatically.  Once you have authenticated:

```
$ kubectl config set-credentials USER_NAME --auth-provider=oidc
$ kubectl config set-credentials USER_NAME --auth-provider-arg=idp-issuer-url=( issuer url )
$ kubectl config set-credentials USER_NAME --auth-provider-arg=client-id=( your client id )
$ kubectl config set-credentials USER_NAME --auth-provider-arg=client-secret=( your client secret )
$ kubectl config set-credentials USER_NAME --auth-provider-arg=refresh-token=( your refresh token )
```
The major downside to this approach is you need the client_secret for your client unless you are using OpenUnison which will issue a session specific client_secret.  If not using OpenUnison, in order for this scheme to work each of the users with access to Kubernetes must know the secret.  This isn't how OIDC is really supposed to work and can open up some security holes.  Also, from a policy standpoint this is another password so it could end up having policy management issues around passwords.

#### Option 2 - Use the --token Option

The kubectl command lets you pass in a token using the --token option.  Simply copy and paste the id_token into this option:

```
$ kubectl --token=eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL21sYi50cmVtb2xvLmxhbjo4MDQzL2F1dGgvaWRwL29pZGMiLCJhdWQiOiJrdWJlcm5ldGVzIiwiZXhwIjoxNDc0NTk2NjY5LCJqdGkiOiI2RDUzNXoxUEpFNjJOR3QxaWVyYm9RIiwiaWF0IjoxNDc0NTk2MzY5LCJuYmYiOjE0NzQ1OTYyNDksInN1YiI6Im13aW5kdSIsInVzZXJfcm9sZSI6WyJ1c2VycyIsIm5ldy1uYW1lc3BhY2Utdmlld2VyIl0sImVtYWlsIjoibXdpbmR1QG5vbW9yZWplZGkuY29tIn0.f2As579n9VNoaKzoF-dOQGmXkFKf1FMyNV0-va_B63jn-_n9LGSCca_6IVMP8pO-Zb4KvRqGyTP0r3HkHxYy5c81AnIh8ijarruczl-TK_yF5akjSTHFZD-0gRzlevBDiH8Q79NAr-ky0P4iIXS8lY9Vnjch5MF74Zx0c3alKJHJUnnpjIACByfF2SCaYzbWFMUNat-K1PaUk5-ujMBG7yYnr95xD-63n8CO8teGUAAEMx6zRjzfhnhbzX-ajwZLGwGUBT4WqjMs70-6a7_8gZmLZb2az1cZynkFRj2BaCkVT3A2RrjeEwZEtGXlMqKJ1_I2ulrOVsYx01_yD35-rw get nodes
```

When using OpenUnison, the ScaleJS Token implementation IdTokenLoader will display your current id_token so you can copy and paste it.  Depending on how long the id_token is available this can get cumbersome quickly.  OpenUnison's OIDC implementation for Kubernetes adds a service that lets you get the current id_token using the refresh_token.  This lets you continuously use an existing session.  It won't generate a new id_token the way the kubectl oidc authenticator will, but it does retrieve the current one.  In order to get a new id_token you have to refresh your ScaleJS Token screen.  Once the session no longer exists, either because its expired or the user logged out the refresh_token will stop providing the id_token.  Your kubectl command would look like:

```
$ kubectl --token=`curl https://host/k8stoken?refresh_token=SDFSDGC... 2>/dev/null` get nodes
```

So long as the user's session is still valid, you won't need to make changes to your kubectl command.  This command does NOT work using the set-credentials option in kubectl.

*Add demo of getting an id token from the refresh token*


### How will groups be stored?

Kubernetes only needs to see a single claim with a list of groups, but how does that claim get generated?  Where is the data stored?  Do you control the datastore?  In most instances if using Active Directory




### No existing Identity Provider

### Existing Identity Provider

### Dedicated Directory

### Read-Only Access Directory
