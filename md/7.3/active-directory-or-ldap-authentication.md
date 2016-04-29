# 2.4.2 Active Directory or LDAP authentication

* [Overview](#overview)
* [Before you start](#before)
* [Create a JAAS configuration file](#createJAAS)
  * [Login context name](#loginContextName)
  * [LdapLoginModule attributes you should set](#attributesToSet)
  * [Values for LdapLoginModule attributes](#attributesValues)
  * [Create / edit the configuration file for your application server](#jaasConfFile)
    * [JBoss](#jaasConfFileJBoss)
    * [Tomcat](#jaasConfFileTomcat)
* [Configurations steps](#configurationSteps)
  * [Changing Bonita BPM authentication service](#bonitaAuthService)
  * [Configure JAAS](#configureJAAS)
    * [JBoss](#jbossJaas)
    * [Tomcat](#tomcatJaas)
* [JAAS configuration files examples](#examples)
* [Known limitations](#limitations)
* [Troubleshooting](#troubleshooting)

**Important notes:**

* This documentation applies to an existing and working Bonita BPM installation
(see the [installation instructions](bonita-bpm-installation-overview.md)).
* In order to have functioning Active Directory/LDAP authentication, the user login (username) must exist
both in the LDAP directory and in the Bonita BPM database (user password is checked against the LDAP server but user
information is read from Bonita BPM database).  
We recommend that you use the [LDAP
synchronizer](ldap-synchronizer.md) to create Bonita BPM users in a Bonita BPM database.

## Overview

Bonita BPM can be configured to perform user authentication against an LDAP server such as Active Directory,
Apache Directory Server, or OpenLDAP.

This type of configuration relies on a specific implementation of the Bonita BPM Engine authentication service that delegates the actual
user name and password verification to a [JAAS](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jaas/JAASRefGuide.md) service
configured with an [LDAP
specific Login Module](http://docs.oracle.com/javase/7/docs/jre/api/security/jaas/spec/com/sun/security/auth/module/LdapLoginModule.md).

## Before you start

In order to configure LDAP successfully, make sure you have the following information:

* LDAP server type: Active Directory (AD), Apache Directory Server, or OpenLDAP
* LDAP server address
* LDAP listening port (e.g. 389 by default)
* Is it possible to build the user distinguished name with user name that the user specifies when logging in?  
For
example, if the user name is: `john.smith` and the user DN is: `CN=John	Smith,CN=Users,DC=MyDomain,DC=com`,
it's not possible to build the DN dynamically.  
But it's possible to do so if the
DN is: `uid=john.smith,ou=people,dc=example,dc=com`.  
If it's not possible to build the DN using the
user name you will need the following extra information:
  * The DN of the LDAP entry under which all users are located (e.g. `CN=Users,DC=MyDomain,DC=com`)
  * The user entry objectClass (the most restrictive one). E.g. usually `user` on AD, `inetOrgPerson`
or `organizationalPerson` for other LDAP servers.
  * The user entry attribute used for authentication (e.g. `userPrincipalName` on AD, value: `john.smith@mydomain.com`
or `uid` on other LDAP servers, value: `john.smith`)
* Does the LDAP server allow anonymous search?
* Does the LDAP server allow search for all users that can possibly log in?
* If search can only be performed by a limited number of "technical" accounts you will need the user name and
password of such an account.

## Create a JAAS configuration file

This section explains how to put together all the LDAP server information you have to create or edit a JAAS
configuration file compatible with your JEE application server.

### Login context name

The JAAS configuration can include one or several login contexts. The Bonita BPM login context must be named
`BonitaAuthentication-`
(where `` is your tenant id).

### LdapLoginModule attributes to set

It's important to identify which
`LdapLoginModule`
attributes you need to set.
This will be at least one of `authIdentity`, `userFilter`, `tryFirstPass`, `java.naming.security.principal` or `java.naming.security.credentials`.
Based on the information described in the "Before you start" section, you can identify which of the following cases
applies:

* If you can build the user DN by directly injecting the user name =\> set only the `authIdentity` attribute
* If you cannot build the DN and anonymous search is allowed =\> set only the `userFilter` attribute
* If you cannot build the DN and anonymous search is disallowed and authenticated users can search =\> set the `userFilter`
and `authIdentity` attributes
* If you cannot build the DN and anonymous search is disallowed and authenticated users cannot search =\> set the `userFilter`,
`authIdentity`, `tryFirstPass`, `java.naming.security.principal` and `java.naming.security.credentials`
attributes

### Values for LdapLoginModule attributes

In this section we explain how to set `LdapLoginModule` attributes values.

**`userProvider`**: set this to
`ldap://:/`.
For example: `ldap://localhost:389/CN=Users,DC=MyDomain,DC=com`

**`userFilter`** (only if needed): the value must be a search request that will find your users in the LDAP server. The search
request can be for example:
`(&(objectClass=user)(userPrincipalName={USERNAME}@mydomain.com))`.
Use an LDAP tool (such as Apache Directory Studio) to validate that the request returns the expected result if you
replace {USERNAME} with an actual username.

**`authIdentity`** (only if needed): there are two cases:  
If you can build the user DN, set the attribute value with the user DN and
`{USERNAME}`
tag. For example
`uid={USERNAME},ou=users,dc=example,dc=com`
.  
If you use a
`userFilter`
and users are allowed to search, set the value with
`{USERNAME}@mydomain.com`
for AD and user the DN (same as above) for other LDAP servers.

**`tryFirstPass`**
(only if needed): set this to
`true`.

**`java.naming.security.principal`**
(only if needed): specify the username (AD) or the user DN (other LDAP servers) of a user that can perform searches on the server.

**`java.naming.security.credentials`**
(only if needed): specify the password of a user that can perform searches on the server.

### Create or edit the configuration file for your application server

**Note:** all configuration files are case sensitive. You can find more examples in the [JAAS
configuration files examples](#examples) section of this page.

#### **JBoss**

Edit the `/standalone/configuration/standalone.xml` file to specify the configuration. Use HTML encoding for any strings in the configuration (for example, a space character is written as %20).

Add the Bonita BPM login context using the JBoss specific syntax just before the `` tag.
Note that `security-domain-name` is in fact the JAAS login context name (e.g. Bonita BPM).

The following example is for a tenant with id 1:
`
`

#### **Tomcat**

On Tomcat, the JAAS configuration file follows the [default JVM syntax](http://docs.oracle.com/javase/7/docs/api/javax/security/auth/login/Configuration.md).
Here is an example of JAAS configuration file:  
`
BonitaAuthentication-1 {
        com.sun.security.auth.module.LdapLoginModule sufficient
        userProvider="ldap://localhost:389/ou=people,dc=example,dc=com"
        authIdentity="uid={USERNAME},ou=people,dc=example,dc=com"
        useSSL=false;
        };
    `

We recommend that you name your JAAS configuration file
`jaas.cfg`
and that you add the file under
`/conf`
folder.

## Configuration steps

### Changing Bonita BPM authentication service

The default Bonita BPM installation comes with an authentication service implementation based on the Bonita BPM Engine database. In
order to activate Active Directory/LDAP authentication the service implementation needs to be changed. To
do this, edit `bonita-tenant-sp-custom.properties`. If you edit this in `bonita-home/engine-server/conf/tenants/template/`, the settings will apply to all new tenants.
For an existing tenant, edit the properties file in `bonita-home/engine-server/conf/tenants/`_``_.

You will need to perform following changes:

* Comment out the `authenticationService` line
* Add this new line: `authentication.service.ref.name=jaasAuthenticationService`

### Configure JAAS

#### **JBoss**

As the JAAS configuration in JBoss is already done in a file that already exists, no further configuration is necessary.

#### **Tomcat**

To define the JAAS configuration file location you need to set a JVM property, `java.security.auth.login.config`.
To do this for a system running Tomcat you need to edit the
`setenv` script provided with Bonita BPM and located in `/bin` folder.

##### For Linux and Mac OS

* Edit this file: `/bin/setenv.sh`
* Locate the line that starts: `#SECURITY_OPTS`
* Uncomment this line, i.e. remove the \# sign and set property value to: `%CATALINA_HOME%\conf\jaas.cfg`.
* Locate the line that starts: `CATALINA_OPTS=`
* Add the tag `${SECURITY_OPTS} ` after the tag `${BONITA_HOME}`

##### For Windows

* Edit this file: `/bin/setenv.bat`
* Locate the line that starts: `rem set SECURITY_OPTS`
* Uncomment it, i.e. remove "rem" keyword and set property value to: `${CATALINA_HOME}/conf/jaas.cfg`
* Locate the line that starts: `set CATALINA_OPTS=`
* Add the tag `%SECURITY_OPTS%` after the tag `%BONITA_HOME%`

## JAAS configuration files examples

**Note:** Remember to remove the debug flag for production.

**Note:** These examples use the JAAS standard syntax (as used by Tomcat). They can easily be adapted to the JBoss XML syntax.

### Active Directory

#### Search allowed for all users

In this example, the user name is john.smith:
`
BonitaAuthentication-1 {
    com.sun.security.auth.module.LdapLoginModule sufficient
    userProvider="ldap://localhost:389/CN=Users,DC=MyDomain,DC=com"
    userFilter="(&(objectClass=user)(userPrincipalName={USERNAME}@mydomain.com))"
    authIdentity="{USERNAME}@mydomain.com"
    debug=true
    useSSL=false;
    };
`

In this example, the user name is john.smith@mydomain.com:
`
BonitaAuthentication-1 {
    com.sun.security.auth.module.LdapLoginModule sufficient
    userProvider="ldap://localhost:389/CN=Users,DC=MyDomain,DC=com"
    userFilter="(&(objectClass=user)(userPrincipalName={USERNAME}))"
    authIdentity="{USERNAME}"
    debug=true
    useSSL=false;
    };
`

#### Search allowed only for a technical users

In this example, the user name is john.smith:
`
BonitaAuthentication-1 {
    com.sun.security.auth.module.LdapLoginModule sufficient
    userProvider="ldap://localhost:389/CN=Users,DC=MyDomain,DC=com"
    userFilter="(&(objectClass=user)(userPrincipalName={USERNAME}@mydomain.com))"
    tryFirstPass=true
    java.naming.security.principal="technical.user@mydomain.com"
    java.naming.security.credentials="technical_user_password"
    debug=true
    useSSL=false;
    };
`

### Other LDAP servers

#### Build the user DN using the user name

`
BonitaAuthentication-1 {
        com.sun.security.auth.module.LdapLoginModule sufficient
        userProvider="ldap://localhost:389"
        authIdentity="uid={USERNAME},ou=grenoble,dc=example,dc=com"
        debug=true
        useSSL=false;
        };
    `

## Known limitations

The Active Directory configuration has been tested in single domain configuration. If you a running with multiple domains
it's likely that the user will have to type a username including domain name when logging in.

## Troubleshooting

If necessary, you can enable JAAS debug mode by editing your configuration file and adding the following line: `
debug=true
`

On Active Directory, a common error code is:
`LDAP: error code 49 - 80090308: LdapErr: DSID-0C0903A9, comment: AcceptSecurityContext error, data 52e, v1db1`.
This error code can have several root causes:

* The user doesn't exist in AD: in the JAAS configuration, verify the user filter and validate it using a tool such as
Apache Directory Studio.
* The username doesn't include the domain name: in the JAAS configuration, make sure that the `authIdentity` value
includes the domain name.
* The user password provided is not correct.