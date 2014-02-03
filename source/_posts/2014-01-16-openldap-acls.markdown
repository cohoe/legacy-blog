---
layout: post
title: "OpenLDAP ACLs"
date: 2014-01-16 14:52
comments: true
sharing: true
categories: Guides
alias: /guides/openldap-acls
---

## Default Access Policy
If you have no ACLs configured, the default access policy looks like this:
```
olcAccess: to *
           by * read
```
Translation: Anyone (authenticated or not) can read everything. BAD!

## Anatomy of an ACL
```plain
olcAccess: <access directive>
<access directive> ::= to <what>
    [by <who> [<access>] [<control>] ]+
```
ACLs (or ```<access directive>```'s) are defined in the ```olcAccess``` attributes of the tree. An ```<access directive>``` defines 
1. ```<what>``` you are giving access to (a set of DNs and attributes)
1. ```<who>``` you are giving access to (a set of DNs or special words that identify certain objects)
1. The level of ```<access>``` you are granting them (such as read, write, etc)
1. Any additional ```<control>``` flow instructions (continue looking for more ACLs, stop processing, etc)

We'll be going into each of these statements in much more detail.

## Flow Control
There are three keywords that you can use to define further ACL processing (aka control flow). They are:
1. ```stop``` (implied)
1. ```continue```
1. ```break```

Stop means stop. Further ACL processing will not occur, so you don't get the chance to gain/lose permissions later on. Since this is the implied option you should never have to specify it unless you want to halt at a specific ```<who>```.
If you want to stop processing at the end of your ACL but keep going through it's ```<who>``` clauses, you can use continue. This lets you grant other people privileges based on this ACL but not continue with parsing other ACLs.
If you want to stop going through the ```<who>```s of your ACL but want to keep going through other ACLs, specify ```<break>```. This lets you stop processing on the current ACL and continue with others.

## Implied Stuff
At the end of every ACL there is an implied clause that restricts access:
```
by * none stop
```
This means that when the end of your ACL is reached, there is no more processing for that ```<what>```. This means that if a ```<who>``` that you care about is not in this rule but appears further down the list, it will never reach that later rule.

# Sources:
* OpenLDAP - Configuring slapd (<a href="http://www.openldap.org/devel/admin/slapdconf2.html#Access Control">http://www.openldap.org/devel/admin/slapdconf2.html#Access Control</a>)
* Zytrax LDAP Ch 6 (<a href="http://www.zytrax.com/books/ldap/ch6/">http://www.zytrax.com/books/ldap/ch6/</a>)
* OpenLDAP FAQ (<a href="http://www.openldap.org/faq/data/cache/454.html">http://www.openldap.org/faq/data/cache/454.html</a>)
