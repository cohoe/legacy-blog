---
layout: post
title: "STARRS Installation Guide"
date: 2013-04-24 11:06
comments: true
sharing: true
categories: Guides
alias: /guides/starrs-install
---
# Introduction
## Purpose
The purpose of this guide is to help an IT administrator install and configure the <a href="http://grantcohoe.com/projects/starrs">STARRS</a> network asset tracking application for a basic site. 

## Audience
The intended audience of this guide is IT administrators with Linux experience (specifically Red Hat) and a willingness to use open-source software.

## Commitment
The install process should take around an hour provided appropriate resources.

# Description
## Definition
STARRS is a self-service network resource tracking and registration web application used to monitor resource utilization (such as IP addresses, DNS records, computers, etc). For more details, view the project description <a href="http://grantcohoe.com/projects/starrs">here</a>.

## Physical
STARRS requires a database host and web server to operate. It is recommended to use a single virtual appliance for all STARRS functionality.

## Process
* The virtual appliance is provisioned (out of scope of this guide)
* Core software is installed
* Dependent software packages are downloaded and installed
* STARRS is acquired, configured, and installed
* The web server is configured to allow access

# Installation
## Virtual Appliance
STARRS was tested only on RHEL-based Linux distributions. Anything RHEL6.0+ is compatible. There are no major requirements for the virtual appliance and a minimal software install will suffice. In this example we will be using a Scientific Linux 6.4 virtual machine. 

## Connectivity
Ensure that you are able to log into your remote system as any administrative user (in this example, root) and have internet access.

```
[root@starrs-test ~]# cat /etc/redhat-release
Scientific Linux release 6.4 (Carbon)
```
## System Security
### Firewall
Firewalls can get in the way of allowing web access to the server. Only perform these steps if you have a system firewall installed and intend on using it. In this example we will use the RHEL default ```iptables```.

1. Add a rule to the system firewall to allow HTTP and HTTPS traffic to the server.
```
iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
```

2. Save the firewall configuration (no restart required)
```
service iptables save
```

NOTE: If you have IPv6 enabled on your system, make sure to apply firewall rules to the IPv6 firewall as well.

### SELinux
Any RHEL administrator has dealt with SELinux at some point in their career. There are system-wide settings that allow/deny actions by programs running on the server. Disabling SELinux is not a solution.

1. Allow Apache/httpd to connect to database engines
```
setsebool -P httpd_can_network_connect=on
setsebool -P httpd_can_network_connect_db=on
```

## Core Software
STARRS heavily depends on the PostgreSQL database engine for operation. PgSQL must be at version 9.0 or higher, which is NOT available from the standard RHELish repositories. PgSQL will also require the PL/Perl and PL/Python support packages (also NOT located in the repos). You will need to add new software repositories to your appliance.

### Utilities
These programs will be needed at some point or another in the installation process. 

1. Install the following packages through yum.
```
yum install cpan wget git make perl-YAML -y
```

1. We will also need the development tools group of packages installed.
```
yum groupinstall "Development tools" -y
```

### PostgreSQL Database Engine
1. On your own computer, open up <a href="http://yum.postgresql.org/">yum.postgresql.org</a> and click on the latest available PostgreSQL Release link. In this case we will be using 9.2. 

1. Locate the approprate repository link for your operating system (Fedora, CentOS, SL, etc) and architecture (i386, x86_64, etc). In this case we will be using ```Scientific Linux 6 - i386```. 

1. Download the package from the link you located onto the virtual appliance into any convenient directory. 
```
wget http://yum.postgresql.org/9.2/redhat/rhel-6-i386/pgdg-sl92-9.2-8.noarch.rpm
```

1. Install the PostgreSQL repository package file that was downloaded with ```yum```. Answer yes if asked for verification.
```
yum install pgdg-sl92-9.2-8.noarch.rpm
```

1. You need to ensure that the base PostgreSQL packages are hidden from future package searches and updated. Add an exclude line to your base OS repository file located in ```/etc/yum.repos.d/```. This will prevent any PgSQL packages from being used out of the base packages.
```
# This example uses the sl.repo file. This will depend on your variant of OS (CentOS-base.repo for CentOS).
[sl]
name=Scientific Linux $releasever - $basearch
...
exclude=postgresql*

[sl-security]
```

1. Install the required PgSQL packages using the Yum package manager.
```
yum install postgresql92 postgresql92-plperl postgresql92-server
```

1. Initialize the PgSQL database server
```
[root@starrs-test ~]# service postgresql-9.2 initdb
Initializing database:                                     [  OK  ]
[root@starrs-test ~]#
```

1. Make PgSQL start on system boot
```
chkconfig postgresql-9.2 on
```

1. Start the PgSQL service
```
[root@starrs-test ~]# service postgresql-9.2 start
Starting postgresql-9.2 service:                           [  OK  ]
[root@starrs-test ~]#
```

1. ```su``` to the Postgres account and assign a password to the postgres user account using the query below. Exit back to root when done. This password should be kept secure!
```
[root@starrs-test ~]# su postgres -
bash-4.1$ psql
could not change directory to "/root"
psql (9.2.4)
Type "help" for help.

postgres=# ALTER USER postgres WITH PASSWORD 'supersecurepasswordhere';
ALTER ROLE
postgres=# \q
bash-4.1$ exit
exit
[root@starrs-test ~]#
```
The STARRS installer requires passwordless access to the postgres account. This is achievable by creating a ```.pgpass``` file in the root home directory. 

1. Create a file at ```~/.pgpass``` like such:
```
localhost:5432:*:postgres:supersecurepasswordhere
localhost:5432:*:starrs_admin:adminpass
localhost:5432:*:starrs_client:clientpass
```
You can replace the passwords with whatever you want. Note the passwords for later.

1. This file should be readable only by the user that created it. Change permissions accordingly.
```
chmod 600 .pgpass
```

1. You now need to allow users to login to the database server from the server itself. Open the ```/var/lib/pgsql/9.2/data/pg_hba.conf``` and change all methods to ```md5``` like so:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

This will enable login from Localhost only. If you need remote access, only allow the specific IP addresses or subnets that you need. Security is good. 

1. Reload the postgres service to bring in the changes
```
service postgresql-9.2 reload
```

1. Create admin and client accounts for STARRS. 
```
[root@starrs-test ~]# psql -h localhost -U postgres
psql (9.2.4)
Type "help" for help.

postgres=# create user starrs_admin with password 'adminpass';
CREATE ROLE
postgres=# create user starrs_client with password 'clientpass';
CREATE ROLE
postgres=# \q
[root@starrs-test ~]#
```

1. Verify that you can log into the database server without being prompted for a password. 
```
[root@starrs-test ~]# psql -h localhost -U postgres
psql (9.2.4)
Type "help" for help.

postgres=# \q

[root@starrs-test ~]#
```
If you get prompted for a password, <b>STOP!</b> You need to have this working in order to proceed. Make sure you typed everything in the file correctly and its permissions are set.

## Dependencies
STARRS has many other software dependencies in order to function. These are mostly Perl modules that extend the capabilities of the language. These modules are (CPAN and package are provided however not all packages may be available):

* Net::IP (perl-Net-IP)
* Net::LDAP (perl-LDAP)
* Net::DNS (perl-Net-DNS)
* Net::SNMP (perl-Net-SNMP)
* Net::SMTP (perl-Mail-Sender)
* Crypt::DES (perl-Crypt-DES)
* VMware::vCloud
* Data::Validate::Domain

NOTE: The first time you run CPAN you will be asked some basic setup questions. Answering the defaults are fine for most installations.

1. Install each of these modules. Some of them are available as packages in yum.
```
yum install perl-Net-IP perl-LDAP perl-Net-DNS perl-Mail-Sender perl-Net-SNMP perl-Crypt-DES -y
cpan -i Data::Validate::Domain
cpan -i VMware::vCloud
```

## Download/Configure STARRS
STARRS comes in two parts: The backend (database) and the Web interface. Each one is stored in it's own repository on Github. You will need to download both in order to use the application. Right now we will focus on the backend.

1. You will need a directory to store the downloaded repos in. I recommend using ```/opt```. Clone (download) the current versions of the repositories using Git into that directory.
```
[root@starrs-test ~]# cd /opt/
[root@starrs-test opt]# git clone https://github.com/cohoe/starrs -q
[root@starrs-test opt]# ls
starrs
[root@starrs-test opt]#
```

1. Open the installer file at ```/opt/starrs/Setup/Installer.pl```. You will need to edit the values in the Settings section of the file to match your specific installation. Example:
```perl
# Settings
my $dbsuperuser = "postgres";
my $dbadminuser = "starrs_admin";
my $dbadminpass = "adminpass";
my $dbclientuser = "starrs_client";
my $dbclientpass = "clientpass";
my $sample = undef;

my $dbhost = "localhost";
my $dbport = 5432;
my $dbname = "starrs";
```

dbsuperuser is the root postgres account (usually just 'postgres').
dbadminuser is the STARRS admin user you created above.
dbclientuser is the STARRS client user you created above.
sample will populate sample data into the database. Set to anything other than ```undef``` to enable sample content.

1. Run the install script
```text
perl /opt/starrs/Setup/Installer.pl
```

A lot of text will flash across the screen. This is expected. If you see errors, then something is wrong and you should revisit the setup instructions.

You can verify that STARRS is functioning by running some simple queries.
```
[root@starrs-test starrs]# psql -h localhost -U postgres starrs
psql (9.2.4)
Type "help" for help.

starrs=# SELECT api.initialize('root');
NOTICE:  table "user_privileges" does not exist, skipping
CONTEXT:  SQL statement "DROP TABLE IF EXISTS "user_privileges""
PL/pgSQL function api.initialize(text) line 19 at SQL statement
    initialize
------------------
 Greetings admin!
(1 row)

starrs=# SELECT * FROM api.get_systems(NULL);
 system_name | owner | group | comment | date_created | date_modified | type | os_name | last_modifier | platform_name | asset | datacenter | location
-------------+-------+-------+---------+--------------+---------------+------+---------+---------------+---------------+-------+------------+----------
(0 rows)

starrs=# \q
[root@starrs-test starrs]#

```

If you see "Greetings admin!" and you can perform the queries without error, then your backend is all set up and is ready for the web interface.

### Apache2/httpd
The STARRS web interface requires a web server to be installed. Only Apache has been tested. If you have a new system then you might not have Apache (or in RHELish, httpd) installed. 

1. If you do not have Apache/httpd installed, install it.
```
yum install httpd php php-pgsql -y
```

1. Navigate to the ```/etc/httpd/conf.d``` directory.

1. Remove the ```welcome.conf``` that exists there.
```
cd /etc/httpd/conf.d/
rm -rf welcome.conf
```

1. Enable httpd to start on boot
```
chkconfig httpd on
```

1. Start the httpd service. (Warning messages are fine, as long as the service starts you should be fine)
```
[root@starrs-test ~]# service httpd start
Starting httpd: httpd: apr_sockaddr_info_get() failed for starrs-test.grantcohoe.com
httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1 for ServerName
                                                           [  OK  ]
[root@starrs-test ~]#
```

### PHP Configuration
A minor change to the system PHP configuration allows the use of short tags for cleaner code. This feature must be enabled in the ```/etc/php.ini``` file. 

1. In the php.ini file, set ```short_open_tag = on```

### Web Interface
You will be cloning the starrs-web into the Apache web root directory to simply deployment. 

1. Change directory to ```/var/www/html/``` and clone the repository. <b>Note the . at the end of the clone command.</b>
```
[root@starrs-test ~]# cd /var/www/html/
[root@starrs-test html]# git clone https://github.com/cohoe/starrs-web -q .
```

1. Copy the ```application/config/database.php.example``` to ```application/config/database.php```

1. Edit the copied/renamed database file and enter your database connection settings. Example:
```php
$db['default']['hostname'] = 'localhost';
$db['default']['username'] = 'starrs_admin';
$db['default']['password'] = 'adminpass';
$db['default']['database'] = 'starrs';
```

1. Copy the ```application/config/impulse.php.example``` to ```application/config/impulse.php```

1. For this web environment, the defaults in this file are fine. In this file you can change which environment variable to get the current user from.

1. Apache needs to be given some special instructions to serve up the application correctly. Create a file at ```/etc/httpd/conf.d/starrs.conf``` with the following contents:
```
<Directory "/var/www/html">
        AllowOverride all
        AuthType basic
        AuthName "STARRS Sample Auth (root:admin)"
        AuthBasicProvider file
        AuthUserFile    "/etc/httpd/conf.d/starrs-auth.db"
        require valid-user
</Directory>
```

1. Since we reference an authenication database, you will need to create this file (```starrs-auth.db```). 
```
htpasswd -b -c /etc/httpd/conf.d/starrs-auth.db root admin
```

1. Restart Apache to apply the changes. (A reload is not sufficient enough)
```
service httpd restart
```

# Testing
At this point you should have a fully functioning STARRS installation. Navigate to your server in a web browser and you should be prompted for login credentials. As we established in the authentication database file, the username is root and the password is admin. If you get the STARRS main page, then success! Otherwise start looking through log files to figure out what is wrong.

Detailed troubleshooting is out of the scope of this guide. System Administrator cleverness is a rare skill, but is the most useful thing when trying to figure out what happened. Shoot me an email if you really feel something is wrong. 
