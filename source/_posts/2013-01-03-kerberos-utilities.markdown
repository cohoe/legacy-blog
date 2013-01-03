---
layout: post
title: "Kerberos Utilities"
date: 2013-01-03 00:36
comments: true
sharing: true
categories: Guides
---
There are a number of useful Kerberos client utilities that can help you when working with authentication services. 
<h2>kinit</h2>
kinit will initiate a new ticket from the Kerberos system. This is how you renew your tickets to access kerberized services or renew service principals for daemons. You can kinit interactively by simply running kinit and giving it the principal you want to init as:
<pre class="brush:plain">
miranda ~ # kinit grant
Password for grant@GRANTCOHOE.COM: 
miranda ~ # 
</pre>
No response is good, and you can view your initialized ticket with klist (discussed later on). 

You can also kinit with a keytab by giving it the path to a keytab and a principal. 
<pre class="brush:plain">
miranda ~ # kinit -k -t /etc/krb5.keytab host/miranda.grantcohoe.com
miranda ~ # 
</pre>
Note that it did not prompt me for a password. That is because it is using the stored principal key in the keytab to authenticate to the Kerberos server.
<h2>klist</h2>
klist is commonly used for two purposes: 1) List your current Kerberos tickets and 2) List the principals stored in a keytab. Running klist without any arguments will perform the first action. 
<pre class="brush:plain">
miranda ~ # klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: grant@GRANTCOHOE.COM

Valid starting     Expires            Service principal
04/18/12 12:44:31  04/19/12 12:44:30  krbtgt/GRANTCOHOE.COM@GRANTCOHOE.COM
	renew until 04/18/12 12:44:31
</pre>
To list the contents of a keytab:
<pre class="brush:plain">
miranda ~ # klist -k -t /etc/krb5.keytab 
Keytab name: WRFILE:/etc/krb5.keytab
KVNO Timestamp         Principal
---- ----------------- -------------------------------------------------------
   2 02/16/12 14:50:12 host/miranda.grantcohoe.com@GRANTCOHOE.COM
   2 02/16/12 14:50:12 host/miranda.grantcohoe.com@GRANTCOHOE.COM
   2 02/16/12 14:50:12 host/miranda.grantcohoe.com@GRANTCOHOE.COM
miranda ~ # 
</pre>
The duplication of the principal names represents each of the encryption types that are stored in the keytab. In my case, I use three encryption types to store my principals. If you support older deprecated enc-types (as Kerberos calls them), you will see more entries here.
<h2>kdestroy</h2>
kdestroy destroys all Kerberos tickets for your current user. You can verify this by doing:
<pre class="brush:plain">
miranda ~ # kdestroy
miranda ~ # klist
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)
miranda ~ # 
</pre>
<h2>ktutil</h2>
The Keytab Utility lets you view and modify keytab files. To start, you need to read in a keytab
<pre class="brush:plain">
ktutil:  rkt /etc/krb5.keytab 
ktutil:  list
slot KVNO Principal
---- ---- --------------------------------------------------------------------
   1    2 host/miranda.grantcohoe.com@GRANTCOHOE.COM
   2    2 host/miranda.grantcohoe.com@GRANTCOHOE.COM
   3    2 host/miranda.grantcohoe.com@GRANTCOHOE.COM
ktutil:  
</pre>
If you want to merge two keytabs, you can repeat the read command and the second keytab will be appended to the list. You can also selectively add and delete entries to the keytab as well. Once you are done, you can write the keytab out to a file.
<pre class="brush:plain">
ktutil:  wkt /etc/krb5.keytab 
ktutil:  quit
miranda ~ # 
</pre>
