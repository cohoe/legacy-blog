---
layout: post
title: "OpenLDAP ACLs"
date: 2014-01-16 14:52
comments: true
sharing: true
categories: Guides
alias: /guides/openldap-acls
---
"LDAP is hard..." says every real sysadmin at some point in their life. But after enough swearing and failed terminal commands, anyone can become a confident OpenLDAP administrator. One thing that I and some of my fellow sysadmins keep getting caught up on is Access Control Lists (ACLs). So I figured I'd do some quality research and explanation on the topic. In this guide we will expore ACLs in-depth and some practical examples and uses of each.

# Conventions & Setup
Throughout this guide I will refer to the following directory tree. Just to make sure we are all on the same page, the structure is explained as follows:
{% imgpopup /images/ldapacl/root.png 80% Directory Structure %}
Objects in the tree are referred to by a Distinguished Name (DN). The root DN of our directory (for the fictional website _example.com_) is represented as a list of Domain Components (DC) as __DC=example,DC=com__. We then organize our tree into three Organizational Units (OU). __OU=users__ will contain all of the user account objects. The __OU=groups__ will contain all of the user groups that exist in the organization. Finally, we will stash all of our service accounts in __OU=apps__. In this sample tree there are two entries per OU. Users (__UID=ted__, __UID=robin__), groups (__CN=bros__, __CN=hoes__), apps (__CN=brocode__, __CN=slapclock__). The CN attribute represents a container. 

I am using Active Directory DN syntax which uses capital letters for the DN attributes (CN vs cn). LDAP is case-insensitive but the caps make things a bit easier to read here. I also use the terms ACL, Control, and Rule interchangably. Usually I am referring to a specific item in the OpenLDAP access policy.

## Default Access Policy
If you have no ACLs configured, the default access policy looks like this:
```
olcAccess: to *
           by * read
```
Translation: Anyone (authenticated or not) can read everything. BAD!

# Access Control Definition
This is the general form of an ACL as provided by the OpenLDAP documentation:
```plain OpenLDAP 2.4 ACL General Form
 <access directive> ::= to <what>
        [by <who> [<access>] [<control>] ]+
    <what> ::= * |
        [dn[.<basic-style>]=<regex> | dn.<scope-style>=<DN>]
        [filter=<ldapfilter>] [attrs=<attrlist>]
    <basic-style> ::= regex | exact
    <scope-style> ::= base | one | subtree | children
    <attrlist> ::= <attr> [val[.<basic-style>]=<regex>] | <attr> , <attrlist>
    <attr> ::= <attrname> | entry | children
    <who> ::= * | [anonymous | users | self
            | dn[.<basic-style>]=<regex> | dn.<scope-style>=<DN>]
        [dnattr=<attrname>]
        [group[/<objectclass>[/<attrname>][.<basic-style>]]=<regex>]
        [peername[.<basic-style>]=<regex>]
        [sockname[.<basic-style>]=<regex>]
        [domain[.<basic-style>]=<regex>]
        [sockurl[.<basic-style>]=<regex>]
        [set=<setspec>]
        [aci=<attrname>]
    <access> ::= [self]{<level>|<priv>}
    <level> ::= none | disclose | auth | compare | search | read | write | manage
    <priv> ::= {=|+|-}{m|w|r|s|c|x|d|0}+
    <control> ::= [stop | continue | break]
```
We will be taking this apart line by line to understand exactly what is going on and what some use cases are.

## What
### Filters
If you want to use search filters in your ACLs, I suggest you read <a href="http://www.zytrax.com/books/ldap/apa/search.html">this</a>. Filters are a great way to dynamically allow access to a set of objects.


## Level
None - No access (typical)
Disclose - reveal existance to anonymous people
Auth - Allow anonymous people to authenticate
Compare - used for compare operations, not really relevant
Search - Let people search for a specific value, but don't report it
Read - View the value (typical)
Write - Change the value or add an attribute (typical)
Manage - Mess with structuralObjectClasses (really mess things up)

http://www.openldap.org/its/index.cgi/Documentation?id=7795;page%3D1;statetype%3D1

## Control
There are three optional keywords that you can use to define further ACL processing (aka control flow). They are ```stop``` (implied at the end of each rule), ```continue```, and ```break```.

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

# Test
```yaml LDIF sample
dn: olcDatabase={-1}frontend,cn=config
changetype: modify
add: olcAccess
olcAccess: {10}to foobar by baz
```

# Todo
https://itservices.stanford.edu/service/directory/aclexamples
