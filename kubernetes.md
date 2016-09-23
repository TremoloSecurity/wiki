# Kubernetes Identity Management

## Introduction and Background

This page gives an in-depth analysis of Kubernetes' various authentication methods and
provides several reference architectures for deploying an identity management solution
with for Kubernetes.

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


## Reference Architectures

### No existing Identity Provider

### Existing Identity Provider

### Dedicated Directory

### Read-Only Access Directory
