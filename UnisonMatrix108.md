Introduction
------------

This page is the certification matrix for Unison 1.0.7.

OS Platforms
------------

This sections lists the platforms that Unison is certified to run on. If a platform is not listed, it does not mean Unison will not work, only that it has not be certified. Note that systems listed under development are only supported as part of the development process and are NOT supported for production use.

### Production

| Vendor                        | Version | Architecture |
|-------------------------------|---------|--------------|
| CentOS Linux                  | 6.x,7.x     | x64          |
| RedHat Linux                  | 6.x,7.x     | x64          |
| Amazon Web Services Linux AMI | 201*.\* | x64          |

### Development

| Vendor                        | Version               | Architecture |
|-------------------------------|-----------------------|--------------|
| CentOS Linux                  | 6.x,7.x                   | x64          |
| RedHat Linux                  | 6.x,7.x                   | x64          |
| Amazon Web Services Linux AMI | 201*.\*               | x64          |
| Microsoft Windows             | 2008r2, 2012, 21012r2 | x64          |
| Apple OSX                     | Mavericks, Yosemite   | x64          |

Directories
-----------

In general Unison supports LDAPv3 directories and databases with JDBC drivers. The below systems have been certified with Unison, however if a system isn't listed it does not mean that Unison will not support it.

| Vendor              | Product          | Version                             | Notes |
|---------------------|------------------|-------------------------------------|-------|
| OpenLDAP            | OpenLDAP         | &lt;=2.4.X                          |       |
| Microsoft           | Active Directory | 2003,2003r2,2008,2008r2,2012,2012r2 |       |
| RedHat              | 389              | &lt;=1.2.x                          |       |
| MySQL               | Oracle           | &lt;=10.1.x                            |       |
| MariaDB               | MariaDB           | &lt;=10.1.x                            |       |
| Amazon Web Services | SimpleDB         | N/A                                 |       |
| Microsoft           | SQL Server       | 2008,2012                           |       |
| Tremolo Security    | Unison           | &lt;= 1.0.8                         |       |
| MongoDB             | MongoDB          | &gt;= 3.x                           | TLS support requires restarts |

Authentication Mechanisms
-------------------------

| Mechanism                              | Description                                                                     | Notes                                                                                     |
|----------------------------------------|---------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Form Login                             | An HTML login form using a JSP page                                             |                                                                                           |
| SAML2                                  | Validates SAML2 assertions                                                      | HTTP-POST for assertion consumer service, HTTP-Redirect for AuthnRequest and SingleLogout |
| Anonymous                              | Provide unauthenticated access                                                  |                                                                                           |
| IWA (Integrated Windows Authentication | Kerberos, tested with Windows 2003+                                             | Supports multiple domains/realms                                                          |
| Certificate Authentication             | Authenticate with an X.509 Certificate                                          | Tested with software certificates and PIV cards                                           |
| Username Only                          | Collect a username without a password                                           | Meant for use with other mechanisms                                                       |
| Banner Acknowledge                     | Provides a way to make a user acknowledge a set of policies prior to logging in |                                                                                           |
| SMS Token Authentication               | Provide a second token via an SMS message                                       | Currently supports Twilio                                                                 |
| Secret Question Authentication         | Provide a second token via "Secret Questions"                                   |                                                                                           |
| Login Service                          | Allows a user to choose from multiple authentication methods                    |                                                                                           |
| OAuth2 Bearer Tokens                   | Accept Unison Last Mile tokens as OAuth2 bearer tokens                          |                                                                                           |
| Just-In-Time Provisioning              | Execute a workflow as part of the authentication process                        |                                                                                           |
| Persistent Cookie                      | Creates a persistent cookie that can short circuit an authentication chain      |                                                                                           |
| Time based One Time Password (TOTP)    | Uses the TOTP standard for multifactor                                          |                                                                                           |
| OpenID Connect                         | Use the OpenID Connect protocol for authentication                              |                                                                                           |

Provisioning Targets
--------------------

Unison in general supports LDAPv3 compliant directories and databases that support JDBC. If a specific target isn't listed it does not mean that it won't work or won't be supported, only that it hasn't been certified.

| Vendor              | Product          | Version                             | Notes |
|---------------------|------------------|-------------------------------------|-------|
| OpenLDAP            | OpenLDAP         | &lt;=2.4.X                          |       |
| Microsoft           | Active Directory | 2003,2003r2,2008,2008r2,2012,2012r2 |       |
| RedHat              | 389              | &lt;=1.2.x                          |       |
| MySQL               | Oracle           | &lt;=10.1.x                            |       |
| MariaDB               | MariaDB           | &lt;=10.1.x                            |       |
| Amazon Web Services | SimpleDB         | N/A                                 |       |
| Microsoft           | SQL Server       | 2008,2012                           |       |
| PostgreSQL          | PostgreSQL       | 9.x                                 |       |
| Tremolo Security    | Unison           | &lt;= 1.0.7                         |       |
| OpenStack           | Keystone         | v3                                  |       |
| OpenShift           | Origin / Enterprise | 3.0+                             |       |
| Red Hat             | FreeIPA / Identity Management | 4.x                    |       |
| MongoDB             | MongoDB          | &gt;= 3.x                           | TLS support requires restarts |
| OpenStack           | Keystone         | 3.x                                 | Tested on RDO |
| OpenShift           | Red Hat          | 3.x                                 | https://hub.openshift.com/primed/23-tremolo-security-inc |

Last Mile Integration
---------------------

| Vendor         | Product            | OS      | Versions     | Notes                               |
|----------------|--------------------|---------|--------------|-------------------------------------|
| Apache         | HTTPD              | Linux   | 2.2, 2.4     | Apache authentication shared obecjt |
| Oracle         | J2EE               | N/A     | 1.4 - 7      | Java Servlet Filter                 |
| Oracle         | Weblogic           | N/A     | 11g (10.3.6) | Identity Asserter                   |
| RedHat / JBoss | Application Server | N/A     | 6,7          | Supports EAP 5.x                    |
| Apache         | Tomcat             | N/A     | 5,6,7,8        | 5/6 - Servlet Filter, 6-8 Valve     |
| Microsoft      | IIS                | Windows | 7.x          | IHttpModule                         |
| OpenID Connect Bearer Token | N/A | N/A | N/A | Any application expecting a bearer token |

SAML2 Systems
-------------

The below systems have been tested with Unison. If a SAML2 system is not listed, it is not unsupported. Unison supports the HTTP-Post profile for the Assertion Consumer Service and HTTP-Redirect for authentication requests and single logout.

| Name                                 | Vendor    | Version       | Identity Provider | Service Provider | Notes                                          |
|--------------------------------------|-----------|---------------|-------------------|------------------|------------------------------------------------|
| Active Directory Federation Services | Microsoft | 2.0, 3.0      | Yes               | Not Tested       |                                                |
| OpenAM                               | ForgeRock | 9.0,10.0,11.0 | Yes               | 9.0 & 10.0       | 11.0 has a known bug that breaks compatability |
| Shiboleth                            | Internet2 | 2.x           | Yes               | Yes              |                                                |
| Oracle Identity Federation           | Oracle    | 11gR1         | Yes               | Yes              |                                                |

OpenID Connect Systems
-------------

The below systems have been tested with Unison. If an OpenID Connect system is not listed, it is not unsupported. Unison supports is compatible with Google authentication services.

| Name                                 | Vendor    | Version       | Identity Provider | Relying Party | Notes                                          |
|--------------------------------------|-----------|---------------|-------------------|------------------|------------------------------------------------|
| Google Authentication | Google | N/A      | Yes               | N/A       |                                                |
| mod_auth_openidc (https://github.com/pingidentity/mod_auth_openidc)                              | PingIdentity | 2.0 | N/A               | Yes       | Tested on CentOS 7 |
| Microsoft.AspNetCore.Authentication.OpenIdConnect                            | Micrososft | 1.1.0           | N/A               | Yes              |                                                |
| Kubernetes           | Kubernetes    | 1.3+         | N/A               | Yes              |                                                |
| KeyCloak             | Red Hat       | 2.2.1+ | Yes | Not Tested |


Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
