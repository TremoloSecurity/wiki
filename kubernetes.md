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


### Tokens

## RBAC

## Reference Architectures

### No existing Identity Provider

### Existing Identity Provider

### Dedicated Directory

### Read-Only Access Directory
