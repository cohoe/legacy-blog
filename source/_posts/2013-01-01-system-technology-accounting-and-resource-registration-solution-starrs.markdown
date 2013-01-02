---
layout: post
title: "System Technology Accounting and Resource Registration Solution (STARRS)"
date: 2013-01-01 20:10
comments: true
categories: Projects
---
# Background
Like most of my projects, it all started with CSH. RIT allocates us two /24 public-facing networks to distribute out to our users. These resources need to have some degree of accounting in the event a user does something stupid (piracy, kiddie-pr0n, etc). RIT handles this with their own internal application, referred to as "start.rit.edu". Fun aside, Start.RIT was coded by an old CSHer, now RIT employee and CSH advisor. He still maintains portions of it today. When the CSH network got to the point of needing our own internal application, Joe Sunday created "start.csh.rit.edu". Similar to it's RIT counterpart, Start.CSH allowed administrators to register machines and distribute resources on the network. Start.CSH also allowed for users to manage firewall rules for their hosts at the border firewall systems. On the backend, it had a script that would automatically generate an ISC-DHCPD config file and load it into the server.

Unfortunately over time, bits of Start began to break. The fact that users could not register their own machines was a major source of irritation for the RTPs (Root-Type-Persons, or Sysadmins). When the network topology was shifted to accomodate some reorganization within RIT, the firewall rule system broke completely. There was no way to clear out old hosts that havent been on the network in years. And after the house DHCP server was compromised (then obliterated by the RTPs for security), the automatic DHCP generation was completely gone. We needed something new.

Starting in the Spring of 2011 I starting to architect a new application to encoumpase the previous functionality and add a lot of new features and enhancements that would benefit us for years to come. Going into the projects, I knew one thing: It had to be based in PostgreSQL. Pg has native datatypes for IP addresses, subnets, and MAC addresses. I know that this would be invaluable to have later on in the project. I also knew that I didnt want to write a daemon that ran on some server.

# Development
At this point in my career, I didn't have a whole lot of database architecting experience. I began to search for a schema creation application to help me architect what I knew was going to be a complex schema. I ran across a service called SchemaBank. (SB is now defunct) It was a schema design webapp that supported Postgres. It also introduced me to two things that would change the course of the application forever: Triggers and Functions. I realized that I could use Pg as the backend daemon to the entire application. For client interaction with the application, I started to develop a set of wrapper API functions that would help impose the application logic on the stored data. While most of this was taken care of in the relations between entities, there were some that could not be expressed as a relation. Fast forward two months, and I had a first revision of the database and the API to start writing an interface for.

At this point I didn't know any sort of web languages other than HTML and CSS. So using a book I won at BarCampRoc on PHP, MySQL, and Javascript (a very good book by the way), I started to learn PHP. Originally I was going to go with a purely non-OOP model for the webapp, however after consulting one of my web-dev friends (Ben Russell), he suggested that I go with an OOP model and to use some sort of framework to avoid having to write a ton of extra code. At this point I went with one that another one of my web-dev friends had used, Codeigniter. So I set to work and started writing classes. During this time, I knew that I needed a new a new dhcpd.conf generation function. One of my friends (Anthony Gargiulo) expressed an interest in helping out with the project, so I tasked him with writing a generation function in Perl. Four months later, I had a working PHP web interface (and he had a working dhcpd.conf generator). At this point, I had everything except a name. I dont really remember a lot about how it came about, but I think I took the first cool-sounding word that came to my head. I was in a Quake 3 Arena mood that day and remembered an old console command (impulse). I took that word and backronymed it to the IP Management Program for Use in Local Server Environments.

# Initial Implementation
When we returned to RIT in the Fall, I set to work deploying IMPULSE. After a few hickups here and there, everything was in place and in use. Users were fairly receptive to it, mainly that they could register their devices themselves. However after some time, a few problems emerged. The web interface became terribly slow with all of the data that people had entered. It wasnt very efficient at allowing users to complete basic tasks and had a few bugs. Clearly some changes needed to occur, but during the course of the year I didn't want to touch it. I spent 6 months writing code on this project, and I really didnt want to work on it again.

# Refresh
My latest co-op offerred me a chance to take to the code once again, this time with a new use case in mind. They wanted something that could manage network registrations for developers in a complex environment, but there were a few major additions that needed to happen. IMPULSE needed to go global. It needeed more specific access control. It needed better device integration. And most importantly, it also needed a new interface. Armed with all the knowlege I have gained over the last year, I set out to work. 2 months later I had a new web interface and a lot of new fixes and enhancements to the core. However the name IMPULSE no longer applied. So I cooked up a new one: STARRS (System Technology Accounting and Resource Registration Solution). And that's presumably what you actually care about.

# Features
STARRS starts off with Datacenters, which correspond to your physical locations. Each datacenter contains Subnets and VLANs. Each datacenter then contains one or more Availability Zones, which are logical partitions of resources within the datacenter. AZs contain ranges of your subnets.

Within datacenters are Systems. A system is a computer, based on a hardware Platform (usually Custom though). A system can have Interfaces which attach to your network. On each interface can be Addresses that are assigned from your Availability Zones. You can get your addresses in a variety of ways (depending on family). DHCP and Static for IPv4, and Autoconf, DHCPv6, and Static for IPv6. (Oh yeah, STARRS does both IPv4 and IPv6). Your addresses are given to you with a set expiration date. If you dont renew before that date (You will get email notifications), it will be automatically deleted. On your addresses, you can create a variety of DNS Records that can be added into managed DNS Zones. Zone DDNS updates are done with Keys that are stored within STARRS. If your address is configured with DHCP, you can be a member of a DHCP Class. Classes, Ranges, and Subnets all have a variety of configuration options that can be set. You can also set options globally. For network systems, you can store SNMP Credentials that STARRS can use to look up Switchport and CAM Table data. Using this you can easily locate your system interfaces in your infrastructure. And of course, you can search for systems that you have configured.

STARRS depends on your existing authentication infrastructure for its user data. It does not track credentials within itself, but relies on the client to provide the user and external sources for privileging. The STARRS web interface gets the username from the web server from configurable PHP variables. It takes the username and calls an API initializing function that sets up your privilege levels. STARRS can get privilege data from one of the following:

* Local Tables
* OpenLDAP
* Active Directory

Users can be global ADMINs and have RW access to everything in the database. They can be USERs who can create/edit/remove their own stuff, but cannot see certain confidential configuration and cannot edit other peoples stuff. People who are Group Admins can edit other peoples stuff in their group. There is also a PROGRAM user-level that allows external applications Read-Only access to user resources. If you don't exist in the privilege source, you will be unable to initialize and do anything in STARRS.

STARRS also has the ability to automatically generate a configuration file for the ISC-DHCPD server based on the resources you configure. It supports key-based DDNS updates to zones, DHCP classes, global/range/subnet/class options, and more. A cronjob runs to periodically generate, download, and reload the server configuration file. Eventually this might change to something more real, but for now this is what works.

Use cases have STARRS working with the following web authentication services:

* Stanford Webauth
* Basic Auth w/ mod_ldap

# Deployment
All of the code is available on Github in two repositories:

* Core/Database
* Web

There is an unofficial and unsupported command-line interface written by a friend of mine (Ryan Brown) located Here.

STARRS is licensed under the MIT license.

# Demo
You can access the demo at STARRS Demo. Username is 'root' and password is 'admin'. The database resets itself every 24 hours at midnight, so don't expect long-term persistence of data.

# Who Should Use This
If you are a service provider to a group of users who consume network resources that you control. You want some way of accounting for what resources are in use and where they are in your network. The use cases of it so far are:

* Computer Science House - Dorm of college students (about 350 systems)
* Engineering Lab Services - Product developers in 4 sites across 3 countries (1000+ systems)
* Enterprise - My personal VM server (30 systems)

# Future
There are a couple of enhancements on deck for the next release (whever that may be). See the respective Github project issues for details. However STARRS will soon be encompasing libvirt support. When you create a system, you can specify it to be a Virtual Machine living on a Host in one of your datacenters. None of this has been worked on, but the idea is for STARRS to be a complete network management solution. Maybe I'll call it CloudSTARRS (like rockstars, but in the cloud haha).
