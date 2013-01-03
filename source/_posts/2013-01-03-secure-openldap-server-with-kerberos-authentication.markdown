---
layout: post
title: "Secure OpenLDAP Server with Kerberos Authentication"
date: 2013-01-03 00:26
comments: true
sharing: true
categories: Guides
---
<h2>Host Configuration</h2>
There are some things we need to set up prior to installing and configuring OpenLDAP. 
<h3>Kerberos</h3>
You will need a working KDC somewhere in your domain. You also need to have the server configured as a Kerberos client (<code>/etc/krb5.conf</code>) and be able to <code>kinit</code> without issue.

There are two principals that you will need in your <code>/etc/krb5.keytab</code>:
<ul>
<li>host/server.example.com@EXAMPLE.COM
<li>ldap/server.example.com@EXAMPLE.COM
</ul>

<h3>saslauthd</h3>
If you do not already have this working, see my guide on how to set it up.

<h2>Packages</h2>
You need the following for our setup:
<ul>
<li>krb5-server-ldap (for the Kerberos schema)
<li>openldap
<li>openldap-clients
<li>openldap-servers
<li>cyrus-sasl-gssapi
<li>cyrus-sasl-ldap
</ul>

<h2>SSL Certificates</h2>
You will need SSL certificates matching the hostname you intend your LDAP server to listen on (ldap.example.com is different than server.example.com). Procure these from your PKI administrator. I place mine in the default directories as shown:
<ul>
<li>Server Public Key - /etc/pki/tls/certs/slapd.pem
<li>Server Private Key - /etc/pki/tls/private/slapd.pem
<li>CA Certificate - /etc/pki/tls/cert.pem
</ul>

<h2>ACLs</h2>
The LDAP user needs to be able to read the private key and keytab you configured earlier. Ensure that it can.
<pre class="brush:plain">
[root@localhost ~]# setfacl -m u:ldap:r /etc/krb5.keytab
[root@localhost ~]# setfacl -m u:ldap:r /etc/pki/tls/private/slapd.pem
</pre>

<h2>OpenLDAP Configuration</h2>
<h3>Directory Cleanup</h3>
By default there are some extra files in the configuration directory. Since we are not using the cn=config method of configuring the server, we are simply going to remove the things we don't want. 
<pre class="brush:plain">
[root@localhost ~]# cd /etc/openldap/
[root@localhost ~]# rm -rf slapd.d ldap.conf cacerts
</pre>
We will be re-creating ldap.conf later on with the settings that we want. 

<h3>Kerberos Schema</h3>
Copy <code>/usr/share/doc/krb5-server-ldap-1.9/kerberos.schema</code> into your schema directory (default: <code>/etc/openldap/schema</code>. This contains all of the definitions for Kerberos-related attributes. 

<h3>Data Directory</h3>
Decide where you want your data directory to be. By default this is <code>/var/lib/ldap</code> Copy over the standard DB_CONFIG file to that directory from <code>/usr/share/openldap-servers/DB_CONFIG.example</code>. This contains optimization information about the structure of the database. 

<h3>Enable LDAPS</h3>
Some systems require you to explicitly enable LDAPS listening. On Red Hat Enterprise Linux-based distributions, this is done by changing <code>SLAPD_LDAPS<code> to YES in <code>/etc/sysconfig/ldap</code>.

<h3>slapd.conf</h3>
Create a <code>/etc/openldap/slapd.conf</code> to configure your server. You can grab a copy of my basic configuration file <a href="http://archive.grantcohoe.com/projects/ldap/slapd_basic.conf">here</a>.
``` text
# Include base schema files provided by the openldap-servers package.
include /etc/openldap/schema/corba.schema
include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/duaconf.schema
include /etc/openldap/schema/dyngroup.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/java.schema
include /etc/openldap/schema/misc.schema
include /etc/openldap/schema/nis.schema
include /etc/openldap/schema/openldap.schema
include /etc/openldap/schema/pmi.schema
include /etc/openldap/schema/ppolicy.schema

# Site-specific schema and ACL includes. These are either third-party or custom.
include /etc/openldap/schema/kerberos.schema

# Daemon files needed for the running of the daemon
pidfile /var/run/openldap/slapd.pid
argsfile /var/run/openldap/slapd.args

# Limit SASL options to only GSSAPI and not other client-favorites. Apparently there is an issue where
# clients will default to non-working SASL mechanisms and will make you angry.
sasl-secprops noanonymous,noplain,noactive

# SASL connection information. The realm should be your Kerberos realm as configured for the system. The
# host should be the LEGITIMATE hostname of this server
sasl-realm EXAMPLE.COM
sasl-host ldap.example.com

# SSL certificate file paths
TLSCertificateFile /etc/pki/tls/certs/slapd.pem
TLSCertificateKeyFile /etc/pki/tls/private/slapd.pem
TLSCACertificateFile /etc/pki/tls/cert.pem

# Rewrite certain SASL bind DNs to more readable ones. Otherwise you bind as some crazy default
# that ends up in a different base than your actual one. This uses regex to rewrite that weird
# DN and make it become one that you can put within your suffix.
authz-policy from
authz-regexp "^uid=[^,/]+/admin,cn=example\.com,cn=gssapi,cn=auth" "cn=ldaproot,dc=example,dc=com"
authz-regexp "^uid=host/([^,]+)\.example\.com,cn=example\.com,cn=gssapi,cn=auth" "cn=$1,ou=hosts,dc=example,dc=com"
authz-regexp "^uid=([^,]+),cn=example\.com,cn=gssapi,cn=auth" "uid=$1,ou=users,dc=example,dc=com"

# Logging
#loglevel 16384
loglevel 256

# Actual LDAP database for things you want
database bdb
suffix "dc=example,dc=com"
rootdn "cn=ldaproot,dc=example,dc=com"
rootpw {MD5}X03MO1qnZdYdgyfeuILPmQ==
directory /var/lib/ldap

# Indicies for the database. These are used to improve performance of the database
index entryCSN eq
index entryUUID eq
index objectClass eq,pres
index ou,cn,mail eq,pres,sub,approx
index uidNumber,gidNumber,loginShell eq,pres

# Configuration database
database config
rootdn "cn=ldaproot,dc=example,dc=com"

# Monitoring database
database monitor
rootdn "cn=ldaproot,dc=example,dc=com"
```

To generate a password for your directory and hash it, you can use the <code>slappasswd</code> utility. See my guide on LDAP utilities for details. After that you should be all set. Start the slapd service, but dont query it yet. 

<h2>Client Configuration</h2>
The <code>/etc/openldap/ldap.conf</code> file is the system-wide default LDAP connection settings. This way you do not have to specify the host and protocol each time you want to run a command. Make it now.
<pre class="brush:plain">
# OpenLDAP client configuration file. Used for host default settings
BASE            dc=example,dc=com
URI             ldaps://ldap.example.com
TLS_CACERT      /etc/pki/tls/cert.pem
TLS_REQCERT     demand
</pre>

<h2>Population</h2>
Right now you should have a running server, but with no data in it. I created a simple example setup file to get a basic tree set up. At this point you need to architect how you want your directory to be organized. You may chose to follow this or chose your own. 
<pre class="brush:plain">
miranda ldap # cat setup.ldif
dn: dc=example,dc=com
dc: example
o: Example Corporation
objectclass: top
objectclass: dcObject
objectclass: organization

dn: ou=hosts,dc=example,dc=com
ou: hosts
objectclass: top
objectclass: organizationalUnit

dn: ou=users,dc=example,dc=com
ou: users
objectclass: top
objectclass: organizationalUnit

dn: cn=ldap1,ou=hosts,dc=example,dc=com
cn: ldap1
sn: ldap1
objectclass: top
objectclass: person

dn: cn=ldap2,ou=hosts,dc=example,dc=com
cn: ldap2
sn: ldap2
objectclass: top
objectclass: person

dn: uid=user,ou=users,dc=example,dc=com
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: posixAccount
objectclass: inetOrgPerson
ou: Users
uid: user
uidNumber: 10000
gidNumber: 150
givenName: Firstname
sn: Lastname
cn: Firstname Lastname
gecos: Firstname Lastname
Description: Firstname Lastname
homeDirectory: /users/user
loginShell: /bin/bash
mail: firstname.lastname@example.com
displayname: Firstname Lastname (user)
userPassword: {SASL}user@EXAMPLE.COM

dn: uid=admin,ou=users,dc=example,dc=com
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: posixAccount
objectclass: inetOrgPerson
ou: Users
uid: admin
uidNumber: 10001
gidNumber: 151
givenName: Firstname
sn: Lastname
cn: Firstname Lastname
gecos: Firstname Lastname
Description: Firstname Lastname
homeDirectory: /users/admin
loginShell: /bin/bash
mail: firstname.lastname@example.com
displayname: Firstname Lastname (admin)
</pre>

You can add the contents of this file to your directory using ldapadd.
<pre class="brush:plain">
ldapadd -x -D cn=ldaproot,dc=example,dc=com -W -f setup.ldif
</pre>

<h2>Testing</h2>
You should now be able to query your server and figure out who it thinks you are based on your Kerberos principal.
<pre class="brush:plain">
[root@ldap ~]# kinit user
Password for user@EXAMPLE.COM:
[root@ldap ~]# ldapwhoami
SASL/GSSAPI authentication started
SASL username: user@EXAMPLE.COM
SASL SSF: 56
SASL data security layer installed.
dn:uid=user,ou=users,dc=example,dc=com
[root@ldap ~]#
</pre>
Here you can see that it thinks that I am <code>uid=user,ou=users,dc=example,dc=com</code> given my principal of <code>user@EXAMPLE.COM</code>.

Now try searching:
<pre class="brush:plain">
[root@ldap ~]# ldapsearch uid=user
SASL/GSSAPI authentication started
SASL username: user@EXAMPLE.COM
SASL SSF: 56
SASL data security layer installed.
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> (default) with scope subtree
# filter: uid=user
# requesting: ALL
#

# user, users, example.com
dn: uid=user,ou=users,dc=example,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: posixAccount
objectClass: inetOrgPerson
ou: Users
uid: user
uidNumber: 10000
gidNumber: 150
givenName: Firstname
sn: Lastname
cn: Firstname Lastname
gecos: Firstname Lastname
description: Firstname Lastname
homeDirectory: /users/user
loginShell: /bin/bash
mail: firstname.lastname@example.com
displayName: Firstname Lastname (user)

# search result
search: 5
result: 0 Success

# numResponses: 2
# numEntries: 1
[root@ldap ~]#
</pre>
