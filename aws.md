Amazon Web Services IDentity as a Service
=============

Introduction
------------

This wiki page will take you step-by-step through creating a private IDentity as a Service (IDaaS) for your application, group or enterprise. This guide can be customized to support your organizations requirements very easily. At the end of this wiki you'll have a solution that can:

1.  Allow users in your enterprise to sign in with their enterprise credentials
2.  Allow you to define workflows that users can request and approvers can act on
3.  View reports on who has requested what access and who approved the access
4.  Begin integrating applications

### Architecture

![Architecture](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/idpaas.png)

The above diagram shows how the IDaaS environment is constructed. On the left of the diagram is Active Directory and Active Directory Federation Services for an enterprise identity provider. Unison will work with almost any SAML2 provider though. If you don't have access to ADFS, or another identity provider, you can use the testing identity provider Tremolo Security's web site at <https://www.tremolosecurity.com/support/> . On the right is the IDaaS environment. One of the Linux images hosts Unison, with the other hosting Scale (on Tomcat). All of the other services an identity management solution needs are provided by AWS services eliminating the need to setup more servers.

When a user accesses the IDaaS:

1.  The user is redirected to their IdP for authentication. The IdP will generate a SAML assertion that includes the attributes we described above.
2.  The user's browser will post the assertion to Unison, which will update the authorization data store with the user's attributes.
3.  Unison will generate a Last Mile token that will be forwarded to Scale.
4.  Scale will retrieve the user's information, as well as any links the user is authorized to see, open requests, etc.

At this point the user can request access, approve access, view reports or use Scale as a jumping point to other applications.

### Data Model

Before creating the IDaaS environment, its important to know which attributes will be managed by the IDaaS. This isn't set in stone, it can easily be changed as your needs change. For the purpose of this document, we will be tracking

1.  givenName
2.  sn
3.  mail
4.  displayName
5.  manager
6.  uid

These names are the LDAP names for first name, last name, email address, display name, manager and login id. Unison is built around an LDAP virtual directory and uses LDAP naming conventions. This list of attributes can be added to at any time. The next step is assembling our prerequisites for the IDaaS solution.

### Prerequisites

The IDaaS setup requires the following before beginning:

1.  2 EC2 Linux instances - This tutorial assumes either Amazon Linux, Red Hat Enterprise Linux 6.x or CentOS 6.x but other flavors of Linux will work too
2.  An AWS account
3.  A service account for Amazon SQS (instructions below)
4.  A relational database - This tutorial uses AuroraDB
5.  JBoss EAP/7.x (for Scale)

Setting Up Unison
-----------------

Unison will be installed on the first EC2 image. The install uses yum to quickly get everything setup.

### Install and Initialize Unison

Install Unison
```bash
$ cd /etc/yum.repos.d
$ wget https://www.tremolosecurity.com/docs/tremolosecurity-docs/configs/tremolosecurity.repo
$ yum install unison`
```

Initialize Unison:

1.  Obtain a license key at <https://www.tremolosecurity.com/support/>
2.  Use the key to initialize Unison by going to <https://host:9090/> on your server

### Firewall Rules

The Unison install does not setup firewall rules for you. Below are a simple set of rules that will allow all traffic in, but will forward port 80 to 8080 and 443 to 8443. Its suggested that you keep Unison in a security group that only allows access to 22 (ssh) and 9090 (the Unison administration console) to trusted network locations.

```
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [1:148]
:OUTPUT ACCEPT [1:148]
-A PREROUTING -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 8080
-A PREROUTING -p tcp -m tcp --dport 443 -j REDIRECT --to-ports 8443
COMMIT
# Completed on Fri Nov 15 14:46:39 2013
# Generated by iptables-save v1.4.7 on Fri Nov 15 14:46:39 2013
*filter
:INPUT ACCEPT [1:52]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [1:148]
COMMIT
```

These rules can be placed in /etc/sysconfig/iptables. Once configured, restart iptables (service iptables restart)

### Proxy Wizard

One Unison is initialized, run the Proxy Wizard on the home screen to initialize user facing services. This will generate a self-signed certificate, create basic rules for username and password authentication and create a logout rule.

### Web Services

To configure Unison for web services:

1.  Go to Manage Certificates
2.  Under SSL Certificates click "Create Certificate"
3.  Fill out the information as appropriate, this certificate will be used to protect all web services calls so configure appropriately
4.  Once the certificate is saved, under Session Keys click "Create Session Key"
5.  For the name use idaas-webservices
6.  Once the key is saved, click on "Manage Web Services"
7.  Next to SSL Certificate choose "webservices"
8.  Next to Session Key choose "idaas-webservices"
9.  Next to Enabled check the box
10. Click "Submit"

Integrating SQS
---------------

Prior to setting up the database, configure SQS. SQS will be used to queue all workflow tasks. This adds two benefits to Unison:

1.  If a task fails because a remote resource is not available, Unison can attempt the task again at a later time.
2.  Unison can scale out by leveraging SQS's scale-ability. When a workflow executes, task 1 may be executed on server 1, task 2 on server 2, task 3 on server 1, etc.

All messages on the queue are encrypted, so if someone gets access to the queues they can't see the task messages without having the decryption keys which are stored in Unison. To integrate SQS:

1.  Login to the AWS console
2.  Open SQS
3.  Create a dead letter queue (ie unisonDLQ)
4.  Create an SMTP queue (ie unisonSMTP)
5.  Assign unisonDLQ as its dead letter queue
6.  Create a task queue (ie unisonTasks)
7.  Assign unisonDLQ as its dead letter queue
8.  Login to Identity and Access Management
9.  Create a new user called unisonSQS
10. Record the user's access key and security key
11.  After the user is created, click on the user and select "Attach Policy"
    -   AmazonSNSRolee
    -   AmaonsSQSFullAccess

12.  Login to the Unison administration console on port 9090
13. Click on "Manage Certificates"
14. Click on "Create Session Key"
15. For a name, specify "queueMessages" (no quotes)
16. Click on "Message Queues"
17. Uncheck "Use Internal Queue"
18. For "Common Configuration" specify the queues created above and the encryption key
19. For "External Queue Configuration" specify com.tremolosecurity.provisioning.jms.providers.AwsSqsConnectionFactory
20. Add the following connection factory parameters

| Name         | Description           |
|--------------|-----------------------|
| awsAccessKey | IAM User's Access Key |
| awsSecretKey | IAM User's Secret Key |

Integrating Aurora
------------------

At this point we're ready to start setting up AuroraDB and integrating it with Unison. AuroraDB will store three databases:

1.  Authorizations and Attributes - This database will provide tables that store the user's attributes and their group memberships. This data will be updated by Unison's provisioning engine and accessed primarily through Unison's integrated LDAP virtual directory.
2.  Audit Database - The audit database stores the information that describes what was requested, who requested it and who approved (or rejected) it as well as any changes to the authorization database.
3.  Quartz Scheduler - Unison integrates the Quartz scheduler for job management. This database is used so the scheduler can run on any Unison instance in the cluster.

### Preparing the Database and Unison

*  Deploy an Aurora Instance
*  Customize the MySQL schema for Unison to include the attributes you want for reporting. For instance if you want to include a user's uid, mail, givenName and sn in reports use the below schema

```sql
CREATE TABLE `allowedApprovers` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `approval` int(11) DEFAULT NULL,
   `approver` int(11) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;


 CREATE TABLE `approvals` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `label` varchar(255) DEFAULT NULL,
   `workflow` mediumtext,
   `workflowObj` text,
   `createTS` datetime DEFAULT NULL,
   `approvedTS` datetime DEFAULT NULL,
   `approver` int(11) DEFAULT NULL,
   `approved` int(11) DEFAULT NULL,
   `reason` text,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `approvers` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `userKey` varchar(255) DEFAULT NULL,
   `sn` varchar(255) DEFAULT NULL,
   `uid` varchar(255) DEFAULT NULL,
   `givenName` varchar(255) DEFAULT NULL,
   `mail` varchar(255) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `auditLogType` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `name` varchar(255) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 insert into auditLogType (name) values ('Add');
 insert into auditLogType (name) values ('Delete');
 insert into auditLogType (name) values ('Replace');

 CREATE TABLE `auditLogs` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `isEntry` int(11) DEFAULT NULL,
   `actionType` int(11) DEFAULT NULL,
   `userid` int(11) DEFAULT NULL,
   `approval` int(11) DEFAULT NULL,
   `attribute` varchar(255) DEFAULT NULL,
   `val` text,
   `workflow` int(11) DEFAULT NULL,
   `target` int(11) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `targets` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `name` varchar(255) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `users` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `userKey` varchar(255) DEFAULT NULL,
   `sn` varchar(255) DEFAULT NULL,
   `uid` varchar(255) DEFAULT NULL,
   `givenName` varchar(255) DEFAULT NULL,
   `mail` varchar(255) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `workflows` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `name` varchar(255) DEFAULT NULL,
   `startTS` datetime DEFAULT NULL,
   `completeTS` datetime DEFAULT NULL,
   `userid` int(11) DEFAULT NULL,
   `requestReason` text,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;

 CREATE TABLE `escalation` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `approval` int(11) DEFAULT NULL,
   `whenTS` datetime DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB;
```

* Run the SQL on your AuroraDB instance
* Create a schema to store authorization data and attributes. Assuming you want to track a users first name, last name, email address, manager, uid and display name use the below schema. This schema can be customized to your needs

```sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `givenName` varchar(255) DEFAULT NULL,
  `sn` varchar(255) DEFAULT NULL,
  `uid` varchar(255) DEFAULT NULL,
  `mail` varchar(255) DEFAULT NULL,
  `manager` varchar(255) DEFAULT NULL,
  `displayName` varchar(255) DEFAULT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB;

CREATE TABLE `groups` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB;

CREATE TABLE `memberships` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user` int(11) NOT NULL,
  `group` int(11) NOT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB;
```

* **OPTIONAL : ** if you decide to fail any user requests that have not been processed in a certain amount of time, create an automatic failure user

```sql
INSERT INTO users (uid,givenName,sn,mail,displayName) values ('noApprovals','noApprovals','noApprovals@company.com','noApprovals','No Approvals');
```
* Create groups for the initial deployment. Once Unison is running you will want to control who can view reports and who can approve of those requests. While you can use any model that works for your organization, for this tutorial run the following SQL:

```sql
INSERT INTO `groups` (name) VALUES ('idm-reporting');
INSERT INTO `groups` (name) VALUES ('approvals-idm-reporting');
INSERT INTO `groups` (name) VALUES ('unison-administrators');
```
* Create accounts for the databases for your audit data and user data
* Upload the mysql jdbc driver to Unison

1.  Login to unison on port 9090
2.  Click on "Manage Proxy"
3.  Click on "Manage Proxy Libraries"
4.  Click on "Choose File" or "Browser" and find the mysql jdbc driver
5.  Click "Save"
6.  Restart Unison

### Configuring the Virtual Directory

1.  Login to the Unison admin system on port 9090
2.  Click on "User Directories"
3.  Under "Create Directory" choose "BasicDB"
4.  Based on the schema above, use the following table to configure the directory

| Option                            |Value                                    |
|-----------------------------|------------------------------------|
| Name                        | entperpriseUsers                   |
| User Directory              | Checked                            |
| Enabled                     | Checked                            |
| Driver                      | com.mysql.jdbc.Driver              |
| URL                         | jdbc:mysql://host:port/usersDBName |
| User                        |                                    |
| Password                    |                                    |
| Validate                    |                                    |
| User Table Name             | users                              |
| User Table Primary Key      | id                                 |
| Connection Validation Query | SELECT 1                           |
| User Groups?                | Checked                            |
| Group Table Name            | groups                             |
| Group Table Primary Key     | id                                 |
| Link Table Name             | memberships                        |
| Link Table User Column      | user                               |
| Link Table group Column     | group                              |

**User Mappings**

| LDAP Name   | DB Column   |
|-------------|-------------|
| uid         | uid         |
| givenName   | givenName   |
| sn          | sn          |
| mail        | mail        |
| displayName | displayName |
| manager     | manager     |

**Group Mappings**

| LDAP Name    | DB Column |
|--------------|-----------|
| cn           | name      |
| uniqueMember | uid       |

* Go to the User Directories screen and click on "Add Insert" at the bottom of the screen and use the below table for the configuration information:

| Option     | Value                                             |
|------------|---------------------------------------------------|
| Name       | addUserRole                                       |
| Class Name | net.sourceforge.myvd.inserts.mapping.AddAttribute |

And for the configuration parameters:

| Option         | Value         |
|----------------|---------------|
| attributeName  | userRole      |
| attributeValue | Users         |
| objectClass    | inetOrgPerson |

Click "Save"

### Configuring the Administrator Directory

Unison provides a separate virtual directory for administrators. For the AWS deployment, we'll want to use the same data so we'll configure the same virtual directory information for the admin directory.

1.  Login to the Unison admin system on port 9090
2.  Click on "Admin Directories"
3.  Under "Create Directory" choose "BasicDB"
4.  Based on the schema above, use the following table to configure the directory

| Option                            | Value                                   |
|-----------------------------|------------------------------------|
| Name                        | entperpriseUsers                   |
| User Directory              | Checked                            |
| Enabled                     | Checked                            |
| Driver                      | com.mysql.jdbc.Driver              |
| URL                         | jdbc:mysql://host:port/usersDBName |
| User                        |                                    |
| Password                    |                                    |
| Validate                    |                                    |
| User Table Name             | users                              |
| User Table Primary Key      | id                                 |
| Connection Validation Query | SELECT 1                           |
| User Groups?                | Checked                            |
| Group Table Name            | groups                             |
| Group Table Primary Key     | id                                 |
| Link Table Name             | memberships                        |
| Link Table User Column      | user                               |
| Link Table group Column     | group                              |

**User Mappings**

| LDAP Name   | DB Column   |
|-------------|-------------|
| uid         | uid         |
| givenName   | givenName   |
| sn          | sn          |
| mail        | mail        |
| displayName | displayName |
| manager     | manager     |

**Group Mappings**

| LDAP Name    | DB Column |
|--------------|-----------|
| cn           | name      |
| uniqueMember | uid       |


### Configuring the Audit Database

1.  Login to Unison's admin system on port 9090
2.  Click on "Manage Certificates"
3.  Under "Session Keys" click "Create Session Key"
4.  For a name, use "workflows" (no quotes) and save
5.  Click on "Approvals"
6.  Assuming you are using the givenName, sn, mail and uid in the reports, use the below table

| Option                           | Value                               |
|----------------------------|--------------------------------|
| Enabled                    | Checked                        |
| Driver                     | com.mysql.jdbc.Driver          |
| URL                        | jdbc:mysql://host:3306/auditdb |
| User Name                  |                                |
| Password                   |                                |
| Validate                   |                                |
| Maximum Connections        | 10                             |
| Maximum Idle Connections   | 10                             |
| Workflow Encryption Key    | workflows                      |
| User Identifier Attribute  | uid                            |
| Report Approver Attributes | uid, givenName, sn, mail       |
| Report User Attributes     | uid, givenName, sn, mail       |
| Mask Attributes            |                                |

Also provide your SMTP settings for SES. **NOTE: you will not be able to test sending an email until AFTER you have configured the queue and SQS.**

### Integrating Quartz

1.  Deploy the MySQL schema for Quartz - <https://www.tremolosecurity.com/docs/tremolosecurity-docs/quartz/tables_mysql_innodb.sql>
2.  Login to Unison on port 9090
3.  Click on Scheduler
4.  Specify a label and an IP mask. The mask is used to determine which IP address should be used for naming this scheduler to guarantee uniqueness in a clustered environment
5.  Check off "Use Database as Store"
6.  Use the below table to configure the database:

| Quartz Database Delegate Class Name | org.quartz.impl.jdbcjobstore.StdJDBCDelegate |
|-------------------------------------|----------------------------------------------|
| Driver                              | com.mysql.jdbc.Driver                        |
| User                                | Database user                                |
| Password                            | XXXXXXX                                      |
| Verify Password                     | XXXXX                                        |
| Max Connections                     | 10                                           |
| Validation Query                    | SELECT 1                                     |

Once the scheduler is setup, create two new queues in SQS, both with unisonDLQ as their dead letter queue:

1.  unisonFixApprovals
2.  unisonAuthFail

-   In the Unison administration system, go to the scheduler and click "Add Job"
-   Choose "Update Authorizations"
-   User the below table:

| Job Name     | updateAZ       |
|--------------|----------------|
| Job Group    | unison         |
| Seconds      | 0              |
| Minutes      | 0              |
| Hours        | Click "All"    |
| Day of Month | Click "All"    |
| Month        | Click "All"    |
| Day of Week  | Click "Ignore" |
| Year         | Click "All"    |

-   After saving, "Queue Name" will appear as an option, specify unisonFixApprovals
-   In the Unison administration system, go to the scheduler and click "Add Job"
-   Choose "Automatically Fail Open Approvals"
-   User the below table:

| Job Name     | autoFailAZ     |
|--------------|----------------|
| Job Group    | unison         |
| Seconds      | 0              |
| Minutes      | 0              |
| Hours        | Click "All"    |
| Day of Month | Click "All"    |
| Month        | Click "All"    |
| Day of Week  | Click "Ignore" |
| Year         | Click "All"    |

-   After saving, use the below table to fill out the new options:

| Queue Name     | unisonAuthFail             |
|----------------|----------------------------|
| Failure Reason | Approval failed escalation |
| Auto Fail User | noApprovals                |

-   After saving the job, go to "Message Queue" in the admin system
-   Click "Add New Listener"
-   Choose "Update Approvals Authorization" and specify unisonFixApprovals for the Queue Name
-   Save
-   After saving the listener, go to "Message Queue" in the admin system
-   Click "Add New Listener"
-   Choose "Automatically Fail Open Approvals" and specify unisonAuthFail for the Queue Name
-   Save

After completing these steps Unison will update approval privileges every hour on the hour for open approvals as well as fail any approvals assigned to "noApprovals"

### Setup SES

1.  Login to the AWS Console
2.  Configure your SES account to send emails
3.  Retrieve your account id and secret key

### Setup Approvals Database

1.  Login to the Unison administration console
2.  Click on "Manage Certificates"
3.  Under Session Keys, click "Create Session Key"
4.  For Name specify "workflows" (no quotes) and Save
5.  Click on "Approvals"
6.  Use the below table to specify options:

| Enabled                    | Checked                                            |
|----------------------------|----------------------------------------------------|
| Driver                     | com.mysql.jdbc.Driver                              |
| URL                        | JDBC URL to the unisonUsers database created above |
| User Name                  |                                                    |
| Password                   |                                                    |
| Validate                   |                                                    |
| Maximum Connections        | 10                                                 |
| Maximum Idle Connections   | 10                                                 |
| Workflow Encryption Key    | workflows                                          |
| User Identifier Attribute  | uid                                                |
| Report Approver Attributes | uid, mail, givenName, sn (all on their own lines)  |
| Report User Attributes     | uid, mail, givenName, sn (all on their own lines)  |
| Mask Attributes            |                                                    |

And for SMTP:

| Host             | email-smtp.us-east-1.amazonaws.com    |
|------------------|---------------------------------------|
| Port             | 587                                   |
| User             | Your user key                         |
| Password         | Your secret key                       |
| Subject          | Open Approvals Require Your Attention |
| From             | Choose an email address               |
| Use SSL          | Checked                               |
| Use SOCKS Proxy  | Unchecked                             |
| SOCKS Proxy Host |                                       |
| SOCKS Proxy Port | 0                                     |
| EHLO Localhost   | localhost                             |

Setup Reports and Workflows
---------------------------

### Setup Organizations

Prior to setting up the reports, the organizations need to be setup:

1.  Click on Organizations
2.  Change Root to the name of your company and click "Save"
3.  Add a child organization called "Identity Management" with NO authorizations rules
4.  Add a second child organization (under your company) called "Identity Management Reporting" with the group name cn=idm-reporting,ou=groups,ou=enterpriseUsers,o=Tremolo

### Import Reports

1.  Click on Reports
2.  Download the reports meant for everyone at <https://www.tremolosecurity.com/docs/tremolosecurity-docs/xml/unison/reports/mysql/iamReportsForAll.xml>
3.  Click "Browse" under Import Reports
4.  Choose iamReportsForAll.xml and click "Import"
5.  Choose both reports and set the organization to Identity Management and click "Import"
6.  Download the reports meant for administrators and auditors at <https://www.tremolosecurity.com/docs/tremolosecurity-docs/xml/unison/reports/mysql/iamAuditReports.xml>
7.  Click "Browse" under Import Reports
8.  Choose iamAuditReports.xml and click "Import"
9.  Choose all reports and set the organization to Identity Management Reporting and click "Import"

### Create Workflows

1.  Download the following workflows
    *  <https://www.tremolosecurity.com/docs/tremolosecurity-docs/xml/unison/workflows/AdminAddUserToGroup.xml>
    *  <https://www.tremolosecurity.com/docs/tremolosecurity-docs/xml/unison/workflows/IDaaSReporting.xml>
2.  For AdminAddUserToGroup
    1.  Click on Workflows
    2.  Paste the contents of the AdminAddUserToGroup.xml into the "Workflow XML" section and set the organization to the root organization
3.  For IDaaSReporting
    1.  Click on Workflows
    2.  Pasted the contents of IDaaSReporting.xml into the "Workflow XML" section and set the organization to "Identity Management"

Setup Admin SSO
---------------

1.  Login to the Unison admin console
2.  Click on Manage Certificates
3.  Under Signature and Encryption Keys, click Create Certificate
4.  Fill in the information to create a certificate for signing all outbound SAML requests
5.  Click on "Manage Admin Service"
6.  For Administrative Constraint Type choose "group"
7.  For Administrative Constraint choose specify cn=unison-administrators,ou=groups,ou=enterpriseUsers,o=Tremolo
8.  Under Administrative Authentication Type choose SAML2
    1.  Under Identity Provider Information / From Metadata click "Using Meta Data"
    2.  Paste in the metadata from your enterprise identity provider
    3.  Under Authentication Information use "Password over SSL" for the Required Authentication Type and <urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified> for the NameID Format
    4.  Under Service Provider Information check "Sign Authentication Requests" and select the key created above for "Authentication Request Signing Key"
    5.  Under Directory Mapping Information keep the defaults.
    6.  Generate metadata and import into your identity provider

9.  Bootstrap access to Unison in your database

```sql
INSERT INTO users (uid) values ('YOURUSERNAME');
```
**_Replace YOURUSERNAME with the username of the user you intend to use to login to Unison_**

```sql
SELECT id FROM users WHERE uid='YOURUSERNAME'
```
**_Get the id of your user_**

```sql
INSERT INTO memberships (\`user\`,\`group\`) VALUES (YOURUSERID,ID\_FOR\_unison-administrators);
```
* **_YOURUDERID is the id field associated with the account you just created_**
* **_ID\_FOR\_unison-administrators is the id in the groups table for the unison-administrators group_**

Finally restart Unison

At this point, when you try to login to the Unison admin console, you should have to authenticate via your SAML2 identity provider

Configure Scale
---------------

Scale, Unison's user interface for the web services exposed by Unison. Scale communicates with Unison via REST like web services that are authenticated using certificate authentication.

#### Running the Application Integration Wizard

At this point the connections to the directories and databases have been configured. When a user logs in Unison needs to:

1.  Validate that the user is authenticated
2.  Get the user's attributes added to the user database
3.  Generate a LastMile token for Scale
4.  Log the user into Scale

The application integration wizard will let us:

1.  Create a SAML2 connection with your corporate identity provider
2.  Run a Just-In-Time Workflow to provision the user into the authorization database
3.  Setup an application configuration for Scale
4.  Configure LastMile for Scale

To get started, login into Unison's administration console and click on the Application wizard

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/1.png)

The first screen is an explanation screen, once you have read it click "Next"

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/2.png)

The Basic Information screen collects information used by the proxy, such as the name of the application, the URL users will put into their browsers and the host of the server running Scale.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/3.png)

Since we've already created a directory based on the authorization database, we don't need to set up a new one now so click "Next" WITHOUT checking "Create new directory?".

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/4.png)

On the next screen we will create a new authentication chain.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/5.png)

Once the page refreshes, choose the SAML2 authentication mechanism:

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/6.png)

After the page refreshes, you'll be prompted with a screen to configure SAML2. Choose to paste in your identity provider's metadata instead of entering it manually.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/7.png)

Paste in your identity provider's metadata:

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/8.png)

Finally on this screen, configure the rest of the information then click "Next". Take note of the "Do Not Attempt to Link to Directory". This keeps Unison from trying to find the user in the directory. The workflow that is generated will load the user after the workflow executes.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/9.png)

After configuring authentication, we will tell Unison that you we are using just-in-time provisioning to create or update the user as you login.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/10.png)

After choosing to use just-in-time provisioning choose the provisioning target we created for the authorization database.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/11.png)

After clicking "Next", add the attribute mappings from the identity provider to our database. If the identity provider has different names for the attributes, set them here.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/12.png)

Finally, configure the LastMile portion. Unison will generate a key.

![Screen](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/13.png)

Once Unison is configured, you can complete the integration with Scale.

#### Deploy Scale

Scale configuration is detailed at <https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/scale/scale-manual-1.0.6.html>. Before getting started:

1.  Get an Amazon Linux EC2 instance running
2.  Install the tomcat8 packages (sudo yum install tomcat8.noarch tomcat8-el-3.0-api.noarch tomcat8-jsp-2.3-api.noarch tomcat8-lib.noarch tomcat8-log4j.noarch tomcat8-servlet-3.1-api.noarch)
3.  Configure Scale per <https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/scale/scale-manual-1.0.6.html>

For the Scale configuration:

```
<?xml version="1.0" encoding="UTF-8"?>
<tns:ScaleConfig xmlns:tns="http://www.tremolosecurity.com/scale" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.tremolosecurity.com/scale scale-config.xsd ">
   <tns:serviceConfiguration keyStorePassword="password" keyStorePath="WEB-INF/scaleKeystore.jks" unisonURL="https://idaas-ws.demo.aws:9093/" lookupAttributeName="uid"/>
   <tns:UiConfig displayNameAttribute="displayName"  homeURL="/scale/">

   </tns:UiConfig>
   <tns:appUiConfig useGenericGroups="true" groupsAttribute="mailx" workflowName="Role">
         <tns:uiDecsionClass className="com.tremolosecurity.scale.testing.TestUIDecisions">
                 <tns:initParams name="allowEdit" value="false"/>
         </tns:uiDecsionClass>
         <tns:frontPage showLinks="true" showLinkOrgs="false">
                 <tns:title>AWS re:Invent Private IDaaS Demo</tns:title>
                 <tns:text>
                         Welcome to your enterprise's IDentity as a Service (IDaaS).  Click on "Request Access" to make role requests.
                 </tns:text>
         </tns:frontPage>
   </tns:appUiConfig>
   <tns:userAttributesConfig>
           <tns:attribute name="displayname" label="Display Name" readOnly="true"
                    />
           <tns:attribute name="uid" label="User Login" readOnly="true" ></tns:attribute>
           <tns:attribute name="givenname" label="First Name" readOnly="true" />
           <tns:attribute name="sn" label="Last Name" readOnly="true"/>
           <tns:attribute name="mail" label="Email Address" readOnly="true"
                     />
           <tns:attribute name="manager" label="Manager Login" readOnly="true"/>
   </tns:userAttributesConfig>
   <tns:workflows saveUserProfileWorkflowName="doesnotmatter" />
   <tns:approvals>
           <tns:attributes>
                   <tns:attribute name="displayname" label="Display Name" />
                   <tns:attribute name="uid" label="User Login" ></tns:attribute>
                   <tns:attribute name="givenname" label="First Name"  />
                   <tns:attribute name="sn" label="Last Name" />
                   <tns:attribute name="mail" label="Email Address"  />
                   <tns:attribute name="manager" label="Manager Login" />
           </tns:attributes>
   </tns:approvals>
</tns:ScaleConfig>
```

Once Scale is deployed, users are now able to login and request access to applications and the IDaaS solution is ready for users to login.

Next Steps
----------

Now that your IDaaS service is up and running, you can begin integrating applications.

### Pattern for Adding Workflows

Most workflows require having an approver. While you can reference this approver directly from Unison, the best practice is to create a group for approvals. To create an approval group, the best approach is to:

1.  Add the group to the groups table in the IDaaS users database
2.  Add users to the group via the AdminAddUserToGroup workflow that we uploaded. This will provide an audit trail of who added the user to the group
3.  When defining your approval steps, choose the group you created as the approver

### Integrating the AWS Console

See our wiki page on integrating Unison and the AWS Console for role management

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
