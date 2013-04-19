---
layout: post
title: "Kerberos Utilities"
date: 2013-01-03 00:36
comments: true
sharing: true
categories: Guides
alias: /guides/kerberosutils
---
There are a number of useful Kerberos client utilities that can help you when working with authentication services. 

# kinit
kinit will initiate a new ticket from the Kerberos system. This is how you renew your tickets to access kerberized services or renew service principals for daemons. You can kinit interactively by simply running kinit and giving it the principal you want to init as:
```
gmcsrvx1 ~ # kinit grant
Password for grant@GRANTCOHOE.COM: 
gmcsrvx1 ~ # 
```
No response is good, and you can view your initialized ticket with klist (discussed later on). 

You can also kinit with a keytab by giving it the path to a keytab and a principal. 
```
gmcsrvx1 ~ # kinit -k -t /etc/krb5.keytab host/gmcsrvx1.grantcohoe.com
gmcsrvx1 ~ # 
```
Note that it did not prompt me for a password. That is because it is using the stored principal key in the keytab to authenticate to the Kerberos server.

# klist
klist is commonly used for two purposes: 1) List your current Kerberos tickets and 2) List the principals stored in a keytab. Running klist without any arguments will perform the first action. 
```
gmcsrvx1 ~ # klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: grant@GRANTCOHOE.COM

Valid starting     Expires            Service principal
04/18/12 12:44:31  04/19/12 12:44:30  krbtgt/GRANTCOHOE.COM@GRANTCOHOE.COM
	renew until 04/18/12 12:44:31
```
To list the contents of a keytab:
```
gmcsrvx1 ~ # klist -k -t /etc/krb5.keytab 
Keytab name: WRFILE:/etc/krb5.keytab
KVNO Timestamp         Principal
---- ----------------- -------------------------------------------------------
   2 02/16/12 14:50:12 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
   2 02/16/12 14:50:12 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
   2 02/16/12 14:50:12 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
gmcsrvx1 ~ # 
```
The duplication of the principal names represents each of the encryption types that are stored in the keytab. In my case, I use three encryption types to store my principals. If you support older deprecated enc-types (as Kerberos calls them), you will see more entries here.

# kdestroy
kdestroy destroys all Kerberos tickets for your current user. You can verify this by doing:
```
gmcsrvx1 ~ # kdestroy
gmcsrvx1 ~ # klist
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)
gmcsrvx1 ~ # 
```

# ktutil
The Keytab Utility lets you view and modify keytab files. To start, you need to read in a keytab
```
ktutil:  rkt /etc/krb5.keytab 
ktutil:  list
slot KVNO Principal
---- ---- --------------------------------------------------------------------
   1    2 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
   2    2 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
   3    2 host/gmcsrvx1.grantcohoe.com@GRANTCOHOE.COM
ktutil:  
```
If you want to merge two keytabs, you can repeat the read command and the second keytab will be appended to the list. You can also selectively add and delete entries to the keytab as well. Once you are done, you can write the keytab out to a file.
```
ktutil:  wkt /etc/krb5.keytab 
ktutil:  quit
gmcsrvx1 ~ # 
```
