---
layout: post
title: "Secure Mirroring Highly-Available OpenLDAP Cluster"
date: 2013-01-03 00:41
comments: true
sharing: true
categories: Guides
alias: /guides/openldap-mirror
---
# Background
My organization stores user information in an OpenLDAP directory server. This includes contact information, shell preferences, iButton IDs, etc. The directory server does not store the users password and relies on a Kerberos KDC to provide authentication via SASL. This is a widely supported configuration and is popular because you can achieve SSO (Single Sign-On) and keep passwords out of your directory. Our shell access machines are configured to do LDAP authentication of users which basically instructs PAM to do a bind to the LDAP server using the credentials provided by the user. If the bind is successful, the user logs in. If not, well sucks to be them. Before this project we had a single server handling the directory running CentOS 5.8/OpenLDAP 2.3. If this server went poof, then all userdata is lost and no one could log into any other servers (or websites if configured with Stanford Webauth. See the guide for more on this). Obviously this is a problem that needed correcting.

# Technology
## LDAP Synchronization
OpenLDAP 2.4 introduced a method of replication called "mirrormode". This allows you to use their already existing syncrepl protocol to synchronize data between multiple servers. Previously (OpenLDAP &lt;= 2.3) only allowed you to do a master-slave style of replication where the slaves are read-only and would be unwilling to perform changes. Obviously this has some limitations in that if your master (or "provider" in OpenLDAP-ish) were to die, then your users cannot do any changes until you get it back online. Using MirrorMode, you can create an "N-way Multi-Master" topology that allows all servers to be read/write and instantly replicate their changes to the others. 

## High Availability
To enable users to have LDAP access even if one of the servers dies, we need a way to keep the packets flowing and automatically failover to the secondary servers. Linux has a package called "Heartbeat" that allows this sort of functionality. If you are familiar with Cisco HSRP or the open-source VRRP, it works the same way. Server A is assigned an IP address (lets say 10.0.0.1) and Server B is assigned a different address (lets say 10.0.0.2). Heartbeat is configured to provide a third IP address (10.0.0.3) that will always be available between the two. On the primary server, an ethernet alias is created with the virtualized IP address. Periodic heartbeats (keep-alive packets) are sent out to the other servers to indicate "I'm still here!". Should these messages start disappearing (like when the server dies), the secondary will notice the lack of updates and automatically create a similar alias interface on itself and assign that virtualized IP. This allows you to give your clients one host, but have it seemlessly float between several real ones. 

## SASL Authentication
The easy way of configuring MirrorMode requires you to store your replication DN's credentials in plaintext in the config file. Obviously this is not very secure since you are storing passwords in plaintext. As such we can use the hosts Kerberos principal to bind to the LDAP server as the replication DN and perform all of the tasks we need to do. This is much better than plaintext!

## SSL
Since we like being secure, we should really be using LDAPS (LDAP + SSL) to get our data. We will be setting this up too using our PKI. We will save this for the end since we want to ensure that our core functionality actually works first. 

# New Servers
I spun up two new machines, lets call them "warlock.example.com" and "ldap2.example.com". Each of them runs my favorite Scientific Linux 6.2 but with OpenLDAP 2.4. You cannot do MirrorMode between 2.3 and 2.4. 

<img src="/sites/default/files/styles/medium/public/ldaptopo.PNG" alt="" title="" class="image-medium" />

# Configuration
## Packages
First we need to install the required packages for all of this nonsense to work. 

* openldap (LDAP libraries)
* openldap-servers (Server)
* openldap-clients (Client utilities)
* cyrus-sasl (SASL daemon)
* cyrus-sasl-ldap (LDAP authentication for SASL)
* cyrus-sasl-gssapi (Kerberos authentication for SASL)
* krb5-workstation (Kerberos client utilities)
* heartbeat (Failover)

If you are coming off of a fresh minimal install, you might want to install openssh-clients and vim as well. Some packages may not be included in your base distribution and may need other repositories (EPEL, etc)

## Kerberos Keytabs
Each host needs its own set of host/ and ldap/ principals as well as a shared one for the virtualized address. In the end, you need a keytab with the following principals:

* host/ldap1.example.com@EXAMPLE.COM
* ldap/ldap1.example.com@EXAMPLE.COM
* host/ldap.example.com@EXAMPLE.COM
* ldap/ldap.example.com@EXAMPLE.COM

WARNING: Each time you write a -randkey'd principal to a keytab, it's KVNO (Key Version Number) is increased, thus invalidating all previous written principals. You need to merge the shared principals into each hosts keytab. See my guide on Kerberos Utilities for information on doing this.

Put this file at /etc/krb5.keytab and set an ACL on it such that the LDAP user can read it. 
```
[root@ldap1 ~]# chmod 640 /etc/krb5.keytab 
[root@ldap1 ~]# setfacl -m u:ldap:r /etc/krb5.keytab 
[root@ldap1 ~]# 
```
If you see something like "kerberos error 13" or "get_sec_context: 13" it means that someone cannot read the keytab, usually the LDAP server. Fix it.

## NTP
Kerberos and the OpenLDAP synchronization require synchronized clocks. If you aren't already set up for this, follow my guide for setting up NTP on a system before continuing. 

## SASL
You need saslauthd to be running on both LDAP servers. If you do not already have this set up, follow my guide for SASL Authentication before continuing. 

## Logging
We will be using Syslog and Logrotate to manage the logfiles for the LDAP daemon (slapd). By default it will spit out to local4. This is configurable depending on your system, but for me I am leaving it as the default. Add an entry to your <code>rsyslog</code> config file (usually <code>/etc/rsyslog.conf</code>) for <code>slapd</code>
```
local4.*                /var/log/slapd.log
```
Now unless we tell <code>logrotate</code> to rotate this file, it will get infinitely large and cause you lots of headaches. I created a config file to automatically rotate this log according the the system defaults. This was done at <code>/etc/logrotate.d/slapd</code>:
```
/var/log/slapd.log {
    missingok
}
```
Then restart <code>rsyslog</code>. Logrotate runs on a cron job and is not required to be restarted.

## Data Directory
Since I am grabbing the data from my old LDAP server, I will not be setting up a new data directory. On the old server and on the new master, the data directory is <code>/var/lib/ldap/</code>. I simply scp'd all of the contents from the old server over to this directory. If you do this, I recommend stopping the old server for a moment to ensure that there are no changes occurring while you work. After scp-ing everything, make sure to chown everything to the LDAP user. I also recommend running <code>slapindex</code> to ensure that all data gets reindexed. 

<code>Client Library Configuration</code>
Now to make it convenient to test your new configuration, edit your <code>/etc/openldap/ldap.conf</code> to change the URI and add the certificate settings. 
```
BASE            dc=example,dc=com
URI             ldaps://ldap.example.com
TLS_CACERT      /etc/pki/tls/example-ca.crt
TLS_REQCERT     allow
```
When configuring this file on the servers, TLS_REQCERT must be set to ALLOW since we are doing Syncrepl over LDAPS. Obviously since we are using a shared certificate for ldap.example.com, it will not match the hostname of the server and will fail. On all of your clients, they should certainly "demand" the certificate. But in this instance that prevents us from accomplishing Syncrepl over LDAPS. 

## Certificates for LDAPS
Since people want to be secure, OpenLDAP has the ability to do sessions inside of TLS tunnels. This works the same way HTTPS traffic does. To do this you need to have the ability to generate SSL certificates based on a given CA. This procedure varies from organization to organization. Regardless of your method, the hostname you chose is critical as this will be the name that is verified. In this setup, we are creating a host called "ldap.example.com" that all LDAP clients will be configured to use. As such the same SSL certificate for both hosts will be generated for "ldap.example.com" and placed on each server.

I placed the public certificate at <code>/etc/pki/tls/certs/slapd.pem</code>, the secret key in <code>/etc/pki/tls/private/slapd.pem</code>, and my CA certificate at <code>/etc/pki/tls/example-ca.crt</code>. After obtaining my files, I verify them to make sure that they will actually work:
```
[root@ldap1 ~]# openssl verify -CAfile /etc/pki/tls/example-ca.crt /etc/pki/tls/certs/slapd.pem
/etc/pki/tls/certs/slapd.pem: OK
[root@ldap1 ~]# 
```

You need to allow the LDAP account to read the private certificate:
```
[root@ldap1 ~]# setfacl -m u:ldap:r /etc/pki/tls/private/slapd.pem
```

## Server Configuration
If you are using a previous server configuration, just scp the entire <code>/etc/openldap</code> code over to the new servers. Make sure you blow away any files or directories that may have been added for you before you copy. If you are not, you might need to do a bit of setup. OpenLDAP 2.4 uses a new method of configuration called "cn=config", which stores the configuration data in the LDAP database. However since this is not fully supported by most clients, I am still using the config file. For setting up a fresh install, see one of my other articles on this. (Still in progress at the time of this writing)

The following directives will need to be placed in your slapd.conf for the SSL certificates:
```
TLSCertificateFile /etc/pki/tls/certs/slapd.pem
TLSCertificateKeyFile /etc/pki/tls/private/slapd.pem
TLSCACertificateFile /etc/pki/tls/example-ca.crt
```
Depending on your system, you may need to configure the daemon to run on ldaps://. On Red-Hat based systems this is in <code>/etc/sysconfig/ldap</code>. 

You need to add two things to your configuration for Syncrepl to function. First to your <code>slapd.conf</code>:
```
# Syncrepl ServerID
serverID 001

# Syncrepl configuration for mirroring instant replication between another 
# server. The binddn should be the host/ principal of this server 
# stored in the Kerberos keytab
syncrepl rid=001
provider=ldaps://ldap2.example.com
type=refreshAndPersist
retry="5 5 300 +"
searchbase="dc=example,dc=com"
attrs="*,+"
bindmethod=sasl
binddn="cn=ldap1,ou=hosts,dc=example,dc=com"
mirrormode TRUE
```
The serverID value must uniquely identify the server. Make sure you change it when inserting the configuration onto the secondary server. Likewise change the Syncrepl RID as well for the same reason. 

Secondly you need need to allow the replication binddn full read access to all objects. This should go in your ACL file (which could be your slapd.conf, but shouldnt be).
```
access to *
        by dn.exact="cn=ldap1,ou=hosts,dc=example,dc=com" read
        by dn.exact="cn=ldap2,ou=hosts,dc=example,dc=com" read
```
WARNING: If you do not have an existing ACL setup, doing just these entries will prevent anything else from doing anything with your server. These directives are meant to be ADDED to an existing ACL.

You should be all set and ready to start the server. 
```
[root@ldap1 ~]# service slapd start
Starting slapd:                                            [  OK  ] 
[root@ldap1 ~]# 
```

Make sure that when you do your SASL bind, the server reports your DN correctly. To verify this:
```
ldap1 ~ # su ldap -s /bin/bash
bash-4.1$ kinit -k -t /etc/krb5.keytab host/ldap1.example.com
bash-4.1$ ldapwhoami -Q
dn:cn=ldap1,ou=hosts,dc=example,dc=com
bash-4.1$
```
That DN is the one that should be in your ACL and have read access to everything.

Since the LDAP user will always need this principal, I recommend adding a cronjob to keep the ticket alive. I wrote a script that will renew your ticket and fix an SELinux permissioning problem as well. You can get it <a href="http://archive.grantcohoe.com/projects/ldap/slaprenew">here</a>. Just throw it into your <code>/usr/local/sbin/</code> directory. There are better ways of doing this, but this one works just as well.
```
0 */2 * * * /usr/local/sbin/slaprenew
```

## Firewall
Dont forget to open up ports 389 and 636 for LDAP and LDAPSSL!

## Searching
After configuring both the master and the secondary servers, we now need to get the data initially replicated across your secondaries. Assuming you did like me and copied the data directory from an old server, your master is the only one that has real data. Start the service on this machine. If all went according to plan, you should be able to search for an object to verify that it works. 
```
[root@ldap1 ~]# kinit user
Password for user@EXAMPLE.COM:
[root@ldap1 ~]# ldapsearch -h localhost uid=user uidNumber -Q -LLL
dn: uid=user,ou=users,dc=example,dc=com
uidNumber: 10046

[root@ldap1 ~]#
```
If you get errors, increase the debug level (-d 1) and work out any issues. 

## Replication
After doing a kinit as the LDAP user described above (host/ldap2.example.com), start the LDAP server on the secondary. If you tail the slapd log file, you should start seeing replication events occurring really fast. This is the server loading its data from the provider as specified in slapd.conf. Once it is done, try searching the server in a similar fashion as above. 
```
[root@ldap2 ~]# tail /var/log/slapd.log
slapd[15759]: syncrepl_entry: rid=001 be_search (0)
slapd[15759]: syncrepl_entry: rid=001 uid=user,ou=Users,dc=example,dc=com
slapd[15759]: syncrepl_entry: rid=001 be_add uid=user,ou=Users,dc=example,dc=com (0)
slapd[15759]: syncrepl_entry: rid=001 LDAP_RES_SEARCH_ENTRY(LDAP_SYNC_ADD)
slapd[15759]: syncrepl_entry: rid=001 inserted UUID 84ba3544-5be8-102b-9a32-4dfcdb785320
slapd[15759]: syncrepl_entry: rid=001 be_search (0)
slapd[15759]: syncrepl_entry: rid=001 uid=user,ou=Users,dc=example,dc=com
slapd[15759]: syncrepl_entry: rid=001 be_add uid=user,ou=Users,dc=example,dc=com (0)
slapd[15759]: syncrepl_entry: rid=001 LDAP_RES_SEARCH_ENTRY(LDAP_SYNC_ADD)
slapd[15759]: syncrepl_entry: rid=001 inserted UUID 84c04164-5be8-102b-9a33-4dfcdb785320
slapd[15759]: syncrepl_entry: rid=001 be_search (0)
```

## Heartbeat
Heartbeat provides failover for configured resources between several Linux systems. In this case we are going to provide high-availability of an alias IP address (10.0.0.3) which we will distribute to clients as the LDAP server (ldap.example.com). Heartbeat configuration is very simple and only requires three files:

<code>/etc/ha.d/haresources</code> contains the resources that we will be failing over. It should be the same on both servers. 
```
ldap1.example.com IPaddr::10.0.0.3/24/eth0:1/10.0.0.255
```
The reason the name of the host above is not "ldap.example.com" is because the entry must be a node listed in the configuration file (below). 

<code>/etc/ha.d/authkeys</code> contains a shared secret used to secure the heartbeat messages. This is to ensure that someone doesnt just throw up a server and claim to be a part of your cluster.
```
auth 1
1 md5 mysupersecretpassword
```

Make sure to chmod it to something not world-readable (600 is recommend).

Finally, <code>/etc/ha.d/ha.cf</code> is the Heartbeat configuration file. It should contain entries for each server in your cluster, timers, and log files. 
```
mcast           eth0 225.0.0.1 694 1 0
auto_failback   on
keepalive 1
deadtime 5
debugfile       /var/log/ha-debug.log
logfile         /var/log/ha.log
logfacility     local0
udp             eth0
node            ldap1.example.com
node            ldap2.example.com
```

After all this is set, start the heartbeat service. After a bit you should see another ethernet interface show up on your primary. To test failover, unplug one of the machines and see what happens!

## DNS Hack
A side affect of having the LDAP server respond on the shared IP address is that it gets confused about its hostname. As such you need to edit your <code>/etc/hosts</code> file to point the ldap.example.com name to each of the actual host IP addresses. In this example, ldap1 is 10.0.0.1 and ldap2 is 10.0.0.2. You should modify your hosts file to include:
```
10.0.0.1 ldap.example.com ldap
10.0.0.2 ldap.example.com ldap
```
The first entry should be the address of the local system (10.0.0.2 should be the first in the case of ldap2).

Your DNS server should be able to do forward and reverse lookups for <code>ldap.example.com</code> going to 10.0.0.3 (the highly available address). 

If you are having issues binding between the servers or binding from clients, it is usually due to DNS problems.

# Testing
If all went according to plan, you should be able to see Sync messages in your slapd log file. 
```
slapd[11059]: slapd starting
slapd[11059]: do_syncrep2: rid=002 LDAP_RES_INTERMEDIATE - REFRESH_DELETE
```

Likewise if you log into a client machine (in my case, I made this the KDC) you should be able to search at only the ldap.example.com host. The others will return errors. 
```
[root@kdc ~]# ldapsearch -Q -LLL uid=user displayName -h ldap.example.com
dn: uid=user,ou=users,dc=example,dc=com
displayName: Firstname Lastname (user)

[root@kdc ~]# ldapsearch -Q -LLL uid=user displayName -h ldap1.example.com
ldap_sasl_interactive_bind_s: Invalid credentials (49)
        additional info: SASL(-13): authentication failure: GSSAPI Failure: gss_accept_sec_context
[root@kdc ~]# ldapsearch -Q -LLL uid=user displayName -h ldap2.example.com
ldap_sasl_interactive_bind_s: Invalid credentials (49)
        additional info: SASL(-13): authentication failure: GSSAPI Failure: gss_accept_sec_context
[root@kdc ~]# 
```
The reason is that the client has a different view of what the server's hostname is. This difference causes SASL to freak out and not allow you to bind. 

# Errors
```
[root@kdc ~]# ldapsearch -h ldap.example.com uid=user
SASL/GSSAPI authentication started
ldap_sasl_interactive_bind_s: Invalid credentials (49)
        additional info: SASL(-13): authentication failure: GSSAPI Failure: gss_accept_sec_context
[root@kdc ~]#
```
On the server reveals the error of:
```
slapd[9888]: GSSAPI Error: Unspecified GSS failure.  Minor code may provide more information (Wrong principal in request)
slapd[9888]: text=SASL(-13): authentication failure: GSSAPI Failure: gss_accept_sec_context
```
This means your DNS is not hacked or functional. Fix it according to the instructions

The LDAP log says:
```
slapd[1901]: do_syncrepl: rid=005 rc -2 retrying
slapd[1901]: slap_client_connect: URI=ldap://ldap2.example.com ldap_sasl_interactive_bind_s failed (-2)
```
and <code>/var/log/messages</code> contains: 
```
GSSAPI Error: Unspecified GSS failure.  Minor code may provide more information (Credentials cache permissions incorrect)
```

You need to change the SELinux context of your KRB5CC file (default is <code>/tmp/krb5cc_$UIDOFLDAPUSER</code> on Scientific Linux) to something the slapd process can read. Since every time you re-initialize the ticket you risk defaulting the permissions, I recommend using my script in your cronjob from above. If someone finds a better way to do this, please let me know!

If after enabling SSL on your client you receive (using -d 1):
```
TLS: can't connect: TLS error -5938:Encountered end of file.
```
Check your certificate files on the SERVER. Odds are you have an incorrect path.

# Summary
You now have two independent LDAP servers running OpenLDAP 2.4 and synchronizing data between them. They are intended to listen on a Heartbeat-ed service IP address that will always be available even if one of the servers dies. You can also do SASL binds to each server to avoid having passwords stored in your configuration files.

If after reading this you feel there is something that can be improved or isnt clear, feel free to contact me! Also you can grab sample configuration files that I used at <a href="http://archive.grantcohoe.com/projects/ldap">http://archive.grantcohoe.com/projects/ldap</a>.
