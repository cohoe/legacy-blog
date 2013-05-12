---
layout: post
title: "Stanford Webauth on Enterprise Linux"
date: 2013-02-13 12:38
comments: true
sharing: true
categories: Guides
alias: /guides/webauth
---
## Background
Lets say you are in a domain, and you wish to access several different web services. How often do you find you have to enter the same username and password over and over again to get to each website? Wouldn't it be nice if you could just enter it once and automagically have access to all the web services you need? Well guess what? You can!

It all works off of MIT Kerberos. Kerberos is a mechanism that centralizes your authentication to one service and allows for Single Sign-On. The concept of SSO is fairly simple: Enter your credentials once and that's it. It save you from typing them over and over again and protects against certain attacks. 

WebAuth was developed at Stanford University and brings "kerberization" to web services. All you need is some modules, a KDC, and a webserver. 

In this guide, we will be utilizing the Kerberos realm/domain name of EXAMPLE.COM. We will also be using two servers:

* gmcsrvx2.example.com: The Webauth WebKDC
* gmcsrvx3.example.com: A secondary web server
* webauth.example.com: A DNS CNAME to gmcsrvx2.example.com

Note that you will need a pre-existing Kerberos KDC in your network. 

There are three components of the WebAuth system. WebAuth typically refers to the component that provides authorized access to certain content. The WebKDC is the process that handles communication with the Kerberos KDC and distributes tickets out to the client machines. The WebLogin pages are the login/logoff/password change pages that are presented to the user to enter their credentials. Certain binary packages provided by Stanford do not contain components of the WebAuth system. This is why we are going to build it from source. 

## Server Setup
### Packages
Each machine you plan to install WebAuth on needs the following packages:

* httpd-devel
* krb5-devel
* curl-devel
* mod_ssl
* mod_fcgid
* perl-FCGI
* perl-Template-Toolkit
* perl-Crypt-SSLeay
* cpan


Enterprise Linux does not include several required Perl modules for this to work. You are going to need to install them via CPAN. If you have never run CPAN before, do it once just to get the preliminary setup working. 

* CGI::Application::Plugin::TT
* CGI::Application::Plugin::AutoRunmode
* CGI::Application::Plugin::Forward
* CGI::Application::Plugin::Redirect


You also need Remctl, an interface to Kerberos. For EL6 we are using remctl 2.11 available from the Fedora 15 repositories.
```
wget http://mirror.rit.edu/fedora/linux/releases/15/Everything/i386/os/Packages/remctl-2.11-12.fc15.i686.rpm
wget http://mirror.rit.edu/fedora/linux/releases/15/Everything/i386/os/Packages/remctl-devel-2.11-12.fc15.i686.rpm
yum localinstall remctl*
```

### Firewall
Open up ports 80 (HTTP) and 443 (HTTPS) since we will be doing web stuff. 

### NTP
Since you will be doing Kerberos stuff, you need a synchronized clock. If you do not know how to do this, see my guide on <a href="http://www.grantcohoe.com/guides/system/ntp">Setting up NTP</a>.

### Kerberos
Your machine needs to be a Kerberos client (meaning valid DNS, krb5.conf, and host prinicpal in its /etc/krb5.keytab). We will be using three keytabs for this application:
#### /etc/krb5.keytab
* host/gmcsrvx2.example.com@EXAMPLE.COM</li>
#### /etc/webauth/webauth.keytab
* webauth/gmcsrvx2.example.com@EXAMPLE.COM</li>
#### /etc/webkdc/webkdc.keytab
* service/webkdc@EXAMPLE.COM</li>
NOTE: If you wrote these in your home directory and cp'd them into their respective paths, check your SELinux contexts. 

### SSL Certificates
You need an SSL certificate matching the hostname of each server (gmcsrvx2.example.com, gmcsrvx3.example.com, webauth.example.com) avaiable for your Apache configuration. 

_IMPORTANT_:You also need to ensure that your CA is trusted by Curl (the system). If in doubt, cat your CA cert into ```/etc/pki/tls/certs/ca-bundle.crt```. You will get really odd errors if you do not do this. 

### Shared Libraries
We will be installing WebAuth into ```/usr/local``` and this need its libraries to be linked in the system. 
```
echo '/usr/local/lib' >> /etc/ld.so.conf.d/locallib.conf
```
We will need to run ```ldconfig``` to load in the WebAuth libraries later on. 

## Compile and Install WebAuth
The latest version at the time of this writing is webauth-4.1.1 and is avaiable from <a href="http://webauth.stanford.edu/download.html">Stanford University</a>. Grab the source tarball since the packages do not include features that you will need in a brand new installation. 
```
[root@gmcsrvx2 ~]# tar xfz webauth-4.1.1.tar.gz
[root@gmcsrvx2 ~]# cd webauth-4.1.1
[root@gmcsrvx2 webauth-4.1.1]# ./configure --enable-webkdc
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /bin/mkdir -p
....
....
config.status: executing depfiles commands
config.status: executing libtool commands
config.status: executing include/webauth/defines.h commands
[root@gmcsrvx2 webauth-4.1.1]# 
```

Assuming everything is all good, go ahead and build it.
```
[root@gmcsrvx2 webauth-4.1.1]# make
make  all-am
make[1]: Entering directory `/root/webauth-4.1.1'
....
....
Manifying blib/man3/WebKDC::WebKDCException.3pm
make[2]: Leaving directory `/root/webauth-4.1.1/perl'
make[1]: Leaving directory `/root/webauth-4.1.1'
[root@gmcsrvx2 webauth-4.1.1]# 
```

And assuming everything is happy, make install:
```
[root@gmcsrvx2 webauth-4.1.1]# make install
make[1]: Entering directory `/root/webauth-4.1.1'
test -z "/usr/local/lib" || /bin/mkdir -p "/usr/local/lib"
....
....
test -z "/usr/local/include/webauth" || /bin/mkdir -p "/usr/local/include/webauth"
 /usr/bin/install -c -m 644 include/webauth/basic.h include/webauth/keys.h include/webauth/tokens.h include/webauth/util.h include/webauth/webkdc.h '/usr/local/include/webauth'
make[1]: Leaving directory `/root/webauth-4.1.1'
[root@gmcsrvx2 webauth-4.1.1]# 
```

Now ensure that the new libraries get loaded by running ```ldconfig```. This looks at the files in that ld.so.conf.d directory and adds them to the library path. MAKE SURE YOU DO THIS!!!

The Apache modules for WebAuth have been compiled into ```/usr/local/libexec/apache2/modules```:
```
[root@gmcsrvx2 ~]# ls /usr/local/libexec/apache2/modules/
mod_webauth.la  mod_webauthldap.la  mod_webauthldap.so  mod_webauth.so  mod_webkdc.la  mod_webkdc.so
```

## mod_webauth Configuration
Make sure you have the proper keytab set up at ```/etc/webauth/webauth.keytab```. This file can technically be located anywhere as long as you configure the module accordingly. 

Next you need to create a local state directory for WebAuth. Usually this is ```/var/lib/webauth```. The path can be changed as long as you set it in the following configuration file.

The Apache module needs a configuration file to be included in the Apache server configuration. On Enterprise Linux, this can be located in ```/etc/httpd/conf.d/mod_webauth.conf```:
```
# Load the module that you compiled.
LoadModule webauth_module     /usr/local/libexec/apache2/modules/mod_webauth.so

# Some fancy WebAuth stuff
WebAuthKeyRingAutoUpdate      on
WebAuthKeyringKeyLifetime     30d

# The path to some critical files. These should be secured. 
WebAuthKeyring                /var/lib/webauth/keyring
WebAuthServiceTokenCache      /var/lib/webauth/service_token_cache
WebAuthCredCacheDir           /var/lib/webauth/cred_cache

# The path to the keytab that webauth will use to authenticate with your KDC
WebAuthKeytab                 /etc/webauth/webauth.keytab

# The URL to point to if you need to login. MAKE SURE THIS IS CORRECT!
WebAuthLoginURL               "https://webauth.example.com/login"
WebAuthWebKdcURL              "https://webauth.example.com/webkdc-service/"

# The Kerberos principal to use when authenticating with the keytab above
WebAuthWebKdcPrincipal        service/webkdc

# SSL is a good thing. Plaintext passwords are bad. Secure this server.
WebAuthSSLRedirect            On
WebAuthWebKdcSSLCertFile      /etc/pki/example.com/example-ca.crt
```
Again, adjust paths to suit your tastes. The ```WebAuthWebKdcSSLCertFile``` should be your public CA certificate that you probably cat'd earlier.

## mod_webkdc Configuration
Make sure you have the proper keytab set up at ```/etc/webkdc/webkdc.keytab```. This file can technically be located anywhere as long as you configure the module accordingly. 

Next you need to create a local state directory for the WebKDC. Usually this is ```/var/lib/webkdc```. The path can be changed as long as you set it in the configuration files.

The WebKDC utilizes an ACL to allow only certain hosts to get tickets. This goes in ```/etc/webkdc/token.acl```:
```
# These lines allow all principals that under the webauth service to generate a token
krb5:webauth/*@EXAMPLE.COM id
krb5:webauth/gmcsrvx2.example.com@EXAMPLE.COM cred krb5 krbtgt/EXAMPLE.COM@EXAMPLE.COM
```

For the WebLogin pages, another configuration file is used to specify the information for it (in ```/etc/webkdc/webkdc.conf```). This is seperate from the Apache module configuration loaded by the webserver.
``` perl
# The KEYRING_PATH should match what you put in your httpd config
$KEYRING_PATH = "/var/lib/webkdc/keyring";
$URL = "https://webauth.example.com/webkdc-service/";
# You can make custom skins for the weblogin page. Change the path here
$TEMPLATE_PATH = "./generic/templates";
```
NOTE: You CANNOT change the location of this file or the WebKDC module will freak out. 

Note that certain directives must match the Apache configuration. We will make that now in ```/etc/httpd/conf.d/mod_webkdc.conf```:
``` text
# Load the module that you compiled.
LoadModule webkdc_module /usr/local/libexec/apache2/modules/mod_webkdc.so

# Some fancy WebKdc stuff
WebKdcServiceTokenLifetime    30d

# The path to some critical files. These should be secured.
WebKdcKeyring                 /var/lib/webkdc/keyring

# The path to the keytab and access control list that the webkdc will use to authenticate with your KDC
WebKdcKeytab                  /etc/webkdc/webkdc.keytab
WebKdcTokenAcl                /etc/webkdc/token.acl

# Debugging information is wonderful. Turn this off when you get everything working.
WebKdcDebug                   On
```
Ensure that your paths match what you set up earlier. 

## mod_webauthldap Configuration
This step should only be done if you have an LDAP server operating in your network and can accept SASL binds. You can make certain LDAP attributes available in the web environment as server variables for your web applications. This is configured in ```/etc/httpd/conf.d/mod_webauthldap.conf```:
``` 
# Load the module that you compiled
LoadModule webauthldap_module /usr/local/libexec/apache2/modules/mod_webauthldap.so

# Webauth Keytab & credential cache file
WebAuthLdapKeytab /etc/webauth/webauth.keytab
WebAuthLdapTktCache /var/lib/webauth/krb5cc_ldap

# LDAP Host Information
WebAuthLdapHost ldap.example.com
WebAuthLdapBase ou=users,dc=example,dc=com
WebAuthLdapAuthorizationAttribute privilegeAttribute
WebAuthLdapDebug on

<Location />
        WebAuthLdapAttribute givenName
        WebAuthLdapAttribute sn
        WebAuthLdapAttribute cn
        WebAuthLdapAttribute mail
</Location>
```

## Misc Permissions
Most of the files that you created do not have the appropriate permissions to be read/written by the webserver. Lets fix that.
```
[root@gmcsrvx2 ~ ]# chcon -t bin_t /usr/local/share/weblogin/*.fcgi
[root@gmcsrvx2 ~ ]# chown -R apache:apache /usr/local/share/weblogin
[root@gmcsrvx2 ~ ]# chcon -R -t httpd_sys_rw_content_t /usr/local/share/weblogin/generic
[root@gmcsrvx2 ~ ]# chown root:apache /etc/webkdc/webkdc.keytab
[root@gmcsrvx2 ~ ]# chmod 640 /etc/webkdc/webkdc.keytab
[root@gmcsrvx2 ~ ]# chown root:apache /etc/webauth/webauth.keytab
[root@gmcsrvx2 ~ ]# chmod 640 /etc/webauth/webauth.keytab
[root@gmcsrvx2 ~ ]# chown -R apache:apache /var/lib/webkdc
[root@gmcsrvx2 ~ ]# chown -R apache:apache /var/lib/webauth
[root@gmcsrvx2 ~ ]# chmod 700 /var/lib/webkdc
[root@gmcsrvx2 ~ ]# chmod 700 /var/lib/webauth
```

## Apache VHosts
I created a VHost for both "webauth.example.com" and "gmcsrvx2.example.com", with the latter requiring user authentication to view. I name these after the hostname they are serving in ```/etc/httpd/conf.d/webauth.conf```, Just throw a simple Hello World into the DocumentRoot that you configure for your testing host. Note that you must have NameVirtualHost-ing setup for both ports 80 and 443. 
```
<VirtualHost *:80>
	ServerName webauth.example.com
	ServerAlias webauth
	# Send them to somewhere useful if they request the root of this VHost
	RedirectMatch   permanent ^/$ https://gmcsrvx2.example.com/
	# Send non-HTTPS traffic to HTTPS since we are dealing with passwords
	RedirectMatch   permanent ^/(.+)$ https://webauth.example.com/$1
</VirtualHost>

<VirtualHost *:443>
	# Name to respond to
	ServerName webauth.example.com
	ServerAlias webauth

	# Root directory
	DocumentRoot /usr/local/share/weblogin

	# SSL
	SSLEngine On
	SSLCertificateFile /etc/pki/example.com/webauth/host-cert.pem
	SSLCertificateKeyFile /etc/pki/example.com/webauth/host-key.pem
	SSLCACertificateFile /etc/pki/example.com/example-ca.crt

	# Web Login directory needs some special love
	<Directory "/usr/local/share/weblogin">
			AllowOverride none
			Options ExecCGI
			AddHandler fcgid-script .fcgi
			Order allow,deny
			Allow from all
	</Directory>

	# This allows you to not need to put the file extension on the scripts
	ScriptAlias /login "/usr/local/share/weblogin/login.fcgi"
	ScriptAlias /logout "/usr/local/share/weblogin/logout.fcgi"
	ScriptAlias /pwchange "/usr/local/share/weblogin/pwchange.fcgi"

	# More special options to make things load right based on your template
	Alias /images "/usr/local/share/weblogin/generic/images"
	Alias /help.html "/usr/local/share/weblogin/generic/help.heml"
	Alias /style.css "/usr/local/share/weblogin/generic/style.css"

	# This is the actual web KDC
	<Location /webkdc-service>
			SetHandler webkdc
	</Location>
</VirtualHost>
```

And the host file at ```/etc/httpd/conf.d/gmcsrvx2.conf```:
```
<VirtualHost *:80>
	ServerName gmcsrvx2.example.com
	ServerAlias gmcsrvx2
	DocumentRoot /var/www/html
	RedirectMatch   permanent ^/(.+)$ https://gmcsrvx2.example.com/$1
</VirtualHost>

<VirtualHost *:443>
	ServerName gmcsrvx2.example.com
	ServerAlias gmcsrvx2
	DocumentRoot /var/www/gmcsrvx2
	SSLEngine On
	SSLCertificateFile /etc/pki/example.com/gmcsrvx2/host-cert.pem
	SSLCertificateKeyFile /etc/pki/example.com/gmcsrvx2/host-key.pem
	SSLCACertificateFile /etc/pki/example.com/example-ca.crt

	# Require a webauth valid user to access this directory
	<Directory "/var/www/gmcsrvx2">
			AuthType WebAuth
			Require valid-user
	</Directory>
</VirtualHost>
```

Once you are all set, start Apache. Then navigate your web browser to the host (http://gmcsrvx2.example.com). This should first redirect you to HTTPS, then bounce you to the WebLogin pages. After authenticating, you should be able to access the server.

## Conclusion
This is by no means a simple thing to get set up. An intimate knowlege how how Kerberos, Apache, and LDAP work is crucial to debugging issues relating to WebAuth. All of my sample configuration files can be found in the <a href="http://archive.grantcohoe.com/projects/webauth/">Archive</a>.
