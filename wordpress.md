Wordpress Configuration and Integration
======================================

This guide covers how to configure Unison to work with Wordpress assuming that you have already configured directories and authentication.

Before You Begin
----------------

Before beginning this integration make sure to have:

1.  Obtained the JDBC driver and connection information for Wordpress' database
2.  Created mappings of user attributes and group memberships
3.  Download the Apache LastMile module for your web server

Prepare Your Database
----------------

For Unison to support provisioning into the Wordpress database a view needs to be created to make it easier for a provisioning target to load the user's data from Wordpress.  Assuming MySQL, use the following view:

```sql
CREATE VIEW `wp_unisonusers` AS
  select `wp_users`.`ID` AS `ID`,
  `wp_users`.`user_login` AS `user_login`,
  `wp_users`.`user_pass` AS `user_pass`,
  `wp_users`.`user_nicename` AS `user_nicename`,
  `wp_users`.`user_email` AS `user_email`,
  `wp_users`.`user_url` AS `user_url`,
  `wp_users`.`user_registered` AS `user_registered`,
  `wp_users`.`user_activation_key` AS `user_activation_key`,
  `wp_users`.`user_status` AS `user_status`,
  `wp_users`.`display_name` AS `display_name`,
  (select `wp_usermeta`.`meta_value` from `wp_usermeta` where ((`wp_usermeta`.`user_id` = `wp_users`.`ID`) and (`wp_usermeta`.`meta_key` = 'first_name'))) AS `first_name`,
  (select `wp_usermeta`.`meta_value` from `wp_usermeta` where ((`wp_usermeta`.`user_id` = `wp_users`.`ID`) and (`wp_usermeta`.`meta_key` = 'last_name'))) AS `last_name`,
  (select `wp_usermeta`.`meta_value` from `wp_usermeta` where ((`wp_usermeta`.`user_id` = `wp_users`.`ID`) and (`wp_usermeta`.`meta_key` = 'use_ssl'))) AS `use_ssl`
  from `wp_users`
```

If you want to manage additional attributes, add them to the view.

Provisioning Target
----------------

### Before Configuration

If you haven't already setup the JDBC drivers in Unison, follow these instructions:

1.  Login to Unison's management system (on port 9090 by default)
2.  Click "Manage Proxy" on the left hand side
3.  Scroll to the bottom of the page and click "Manage Proxy Libraries"
4.  Next to "Library" click the browser or choose button depending on your browser and upload the JDBC drivers for your database
5.  Click "Save"
6.  Restart Unison from the command line

### Configuration Steps

1.  Login to the Unison management system (on port 9090 by default)
2.  Click on "Provisioning Targets"
3.  Click on "Add Provisioning Target"
4.  For the name specify Drupal, for the Class Name choose "Relational Database"
5.  Click "Submit"
6.  Fill in the configuration with the below data

#### Target Configuration
| Option                            | Value                                    |   
|-----------------------------------|------------------------------------------|
| Name                              | Wordpress                                   |
| Class Name                        | Relational Database                      |
| Driver                            | *Choose the driver for your database*'   |
| URL                               | *Connection url for your database*       |
| Begin Escape Character (Optional} | *Blank*                                  |
| End Escape Character (Optional)   | *Blank*                                  |
| User Name                         | *User with write access to the database* |
| Password                          | *Account password*                       |
| Verify                            | *Verify the password*                    |
| Maximum Connections               | 10                                       |
| Maximum Idle Connections          | 10                                       |
| User Table                        |                                     |
| User SQL                          | select %S from wp_unisonusers WHERE user_login=%L                                         |
| User Table Primary Key Field      | ID                                      |
| User Table Login Name Field       | user_login                               |
| Group Management Mode             | Custom                               |
| Group Table Name                  |                                      |
| Group SQL                         | select %S from wp_users inner join wp_usermeta on wp_usermeta.user_id=wp_users.id AND wp_usermeta.meta_key='wp_capabilities' where wp_users.id=%I                                         |
| Group Table Primary Key           |                                       |
| Group Table Name Field            | meta_value                                |
| Group Link Table Name             |                              |
| Group Link User Field             | user_id                                      |
| Group Link Group Field            |                                       |
| Custom Provider                   | com.tremolosecurity.unison.provisioning.providers.ext.WordpressProvider     |

#### Target Attribute Map

| Name     | Type   | Value             |
|----------|--------|-------------------|
| first_name     | user   | first_name               |
| last_name     | user   | last_name              |
| display_name      | user   | display_name          |
| user_nicename	   | user | user_nicename                 |
| user_email | user | user_email |
| use_ssl   | static | 0 |

## Workflows

Workflows will depend on your business needs. Assuming you are using Just-In-Time provisioning from a SAML2 assertion, use the below steps to create a basic workflow:

1.  Login to the Unison management portal (port 9090 by default)
2.  Click on "Workflows"
3.  In the box that says "Workflow XML" paste the below XML (or customize it)
5.  Click "Import XML"

```
<workflow name="wp-jit" label="" description="" inList="false" orgid="687da09f-8ec1-48ac-b035-f2f182b9bd1e">
        <mapping strict="true">
          <map>
            <mapping targetAttributeName="user_email" targetAttributeSource="mail" sourceType="user"/>
            <mapping targetAttributeName="first_name" targetAttributeSource="givenName" sourceType="user"/>
            <mapping targetAttributeName="last_name" targetAttributeSource="sn" sourceType="user"/>
            <mapping targetAttributeName="display_name" targetAttributeSource="displayName" sourceType="user"/>
            <mapping targetAttributeName="user_nicename" targetAttributeSource="displayName" sourceType="user"/>
            <mapping targetAttributeName="roles" targetAttributeSource="roles" sourceType="user"/>
          </map>

          <customTask className="com.tremolosecurity.provisioning.customTasks.Attribute2Groups">
            <!-- The name of the attribute to get the group values from. Once the values are added, the attribute is removed from the user.               -->
            <param name="attributeName" value="roles"/>
          </customTask>

          <provision sync="true" target="Wordpress" setPassword="false"/>
        </mapping>
      </workflow>
```

Once the workflow is created, add it using the Just-In-Time provisioning authentication mechanism to your SAML chain.

Proxy Integration
-----------------

Once the directory and provisioning steps are completed Drupal can be integrated into Unison.

### Before Configuration

#### In Unison

Before integrating Drupal, create an authentication chain. If Drupal is running in the cloud its recommended to create a SAML2 chain. See the [TestingIdP](#!testingidp.md) page for how to create a connection to the TremoloSecurity.com testing identity provider.

### Configuring Unison

1.  Login to the Unison Management Portal (port 9090 by default)
2.  Click on "Applications"
3.  Click on "Add Application"
4.  Use the below data

| Option                                | Value                      |
|---------------------------------------|----------------------------|
| Name                                  | Wordpress                     |
| Type                                  | User Application           |
| Session Cookie                        | tremolosession             |
| Session Cookie Secure                 | Checked                    |
| Session Inactivity Timeout (Seconds)  | 900                        |
| Session Cache Timeout in Milliseconds | 3000                       |
| Cookie Domain                         | *The host name for Unison* |
| Logout URI                            | /user/logout               |
| Session Key Alias                     | tremolosession             |

Finally, click "Submit". When the page reloads there will be heading called "URLs". Under this heading click "Add URL" and use the below table to fill in the information:

| Option                        | Value                                                                                            |
|-------------------------------|--------------------------------------------------------------------------------------------------|
| URI                           | /                                                                                                |
| Regular Expression            | Unchecked                                                                                        |
| Proxy To Application          | Checked                                                                                          |
| Proxy To                      | http://host.for.wordpress.server:port${fullURI}                                                     |
| Authentication Chain          | Choose the chain created in "Before Configuration"                                             |
| Authentication Success Result |                                                                                                  |
| Authentication Failure Result | Default Login Failure                                                                            |
| Authorization Success Result  |                                                                                                  |
| Authorization Failure Result  | Default Login Failure                                                                            |



Click "Submit". After the page reloads there will be three new subheadings on the page. For "Hosts", add a new host that matches the host for Unison. This should lineup to the Cookie Domain in the application configuration. For "Rules", create a single rule with an LDAP Scope of "dn" and a Constraint of "o=Tremolo". For Filters, add:

1.  Hide Cookies from Client
4.  Last Mile Security *Use the below options*
    -   Encryption Key - Blank
    -   Specify New Encryption Key - wordpress
    -   Time Skew - 10000
    -   Header Name - tremoloHeader
    -   Post Validation Class Name - *Blank*
    -   Verify Only - Unchecked
    -   Attribute Mapping
        -   uid --&gt; uid *Check User Identifier*
    -   Create Headers - Checked
    -   Keystore Path - *Blank*
    -   Ignore URI - *Blank*
    -   Filter Type - Apache

Create a second URL for /wp-login.php with the same configuration as /, but add WPLogin filter to trigger the login process.

#### In Wordpress

1. Deploy mod_auth_tremolo.so into your Apache
2. Insert your Apache LastMile configuration into the .htaccess file
3. Download and setup the Wordpress cli (http://wp-cli.org/)
4. Go to /var/www/html and run `wp search-replace 'http://installed.host.com' 'https://unison.host.com'` to make sure all URLs served by Wordpress goes through the reverse proxy
5.  Restart Apache and access Wordpress through the reverse proxy, login using the wordpress admin
6.  Install the HTTP Authentication plugin (https://danieltwc.com/2011/http-authentication-4-0/)
7.  Logout of wordpress



Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity

