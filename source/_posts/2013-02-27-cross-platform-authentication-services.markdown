---
layout: post
title: "Cross-Platform Authentication Services"
date: 2013-02-27 22:47
comments: true
sharing: true
categories: Projects
alias: /projects/adkrb
---
You are a nerd. You like doing nerd-y things. You run an entirely \*nix-based network. But like all nerds, you have that one pesky Windows machine that you want to have available for your users. Or you just want the same SSO password from your Kerberos KDC. Regardless, you want Windows to talk to MIT Kerberos. Believe it or not, this is SUPPORTED!

## Process Overview
<img src="http://www.grantcohoe.com/images/blog/cross-auth-process.png" alt="" title=""  />
<br>
We start with the user entering their credentials. These are entered in the standard Windows logon screen (which I dearly miss from Windows 2003). From there, the client is configured to have it's default domain (realm in this case) to be the MIT Kerberos realm that your machine is a member of. This would be the equivalent of an Active Directory domain. 

From here, your workstation acts just like any other Kerberized host. It uses it's host principal (derived from what it thinks its hostname is) and configured password to authenticate to the KDC. Once it has verified it's identity, it then goes to authenticate you. And if your password is correct, Windows lets you in!

The question is: who does it let you in as? A Kerberos principal is nothing more than a name. It is not an account, or any object that actually contains information for the system. You must map Kerberos principals to accounts. These accounts are created in Windows as either local users or via Active Directory (a whole other can of worms). Usually, you will want to create a one-to-one mapping between principal and account (```grant@GRANTCOHOE.COM``` principal = Windows account "grant"). 

## KDC Setup
On your KDC, open up ```kadmin``` with your administrative principal and create a host principal for the new machine. It needs to have a password that you know, so use your favorite password generating source.

```
miranda ~ # kadmin -p cohoe/admin
Authenticating as principal cohoe/admin with password.
Password for cohoe/admin@GRANTCOHOE.COM:
kadmin:  ank host/caprica.grantcohoe.com@GRANTCOHOE.COM
WARNING: no policy specified for host/caprica.grantcohoe.com@GRANTCOHOE.COM; defaulting to no policy
Enter password for principal "host/caprica.grantcohoe.com@GRANTCOHOE.COM":
Re-enter password for principal "host/caprica.grantcohoe.com@GRANTCOHOE.COM":
Principal "host/caprica.grantcohoe.com@GRANTCOHOE.COM" created.
kadmin:  quit
miranda ~ #
```

That's it for the KDC. On to the Windows machine!
## Windows 7 Client
Windows 7 (as well as Server 2008 I believe) include the basic utilities required to support MIT Kerberos. This was available for XP and Server 2003 as "Microsoft Support Tools". The utility we will be using is called ```ksetup```. It allows for configuration of foreign-realm authentication systems. 

So to set your machine to authenticate to your already existing MIT Kerberos KDC, open up a command prompt and do the following:
```
ksetup /setdomain GRANTCOHOE.COM
ksetup /addkdc GRANTCOHOE.COM kerberos.grantcohoe.com
ksetup /setmachpassword passwordfromearlier
ksetup /mapuser * *
```

After that, you need to set a Group Policy setting to automatically pick the right domain for you to login to. Open up the Group Policy Editor (gpedit.msc) and navigate to Local Computer Policy->Computer Configuration->Administrative Templates->System->Login->Assign a default domain for logon. Set this to your realm (GRANTCOHOE.COM in my case). 

Do a nice reboot of your system, and you should be ready to go!

## Active Directory
Active Directory can be configured to trust a foreign Kerberos realm. It will NOT synchronize information with anything, but if you just need a bunch of users to log into something and no data with it, the process is not too terrible. Rather than duplicate the information, you can find the guide that I used here: <a href="http://pig.made-it.com/kerberos-trust.html">Microsoft trusting MIT Kerberos</a> 
