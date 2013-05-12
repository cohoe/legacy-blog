---
layout: post
title: "CSH Wireless Network Deployment"
date: 2013-04-12 17:58
comments: true
sharing: true
categories: Projects
alias: /projects/cshwireless
---
We wanted to deploy our own wireless network across the floor. I decided to make the project happen.

# Requirements
## Spectrum Use
**We can only use three channels in the 5 GHz spectrum.** RIT has a massive unified wireless deployment across all of campus. Policy states that third-parties (students, staff, etc) cannot operate wireless access points within range of the RIT network to prevent signal overlap and interference. This is not an issue in the 5 GHz spectrum, where there are many more non-overlaping channels. This limits potential clients to only those with dual-band wireless radios.

## Authentication
**No client-side software can be required.** This is a requirement set by us to allow ease of access by users. Most operating systems support PEAP/MsCHAPv2 (via RADIUS) out of the box and require no extra configuration to make them work. Unfortunately this requires a bit more system infrastructure to make it work. Our authentication backend (MIT Kerberos) does not support this type of user authentication so we need to do some hax to make it.

## Speed
**We need to be faster than RIT.** There is absolutely no reason to use CSH wireless over RITs unless we can be faster. By using Channel Bonding in two key areas, we can achieve this requirement and double the speed of RIT wireless. We also need to be able to do fast-reassociation between access points so that users can walk up and down floor and not lose connectivity.

# Setup
## Access Points
First we had to figure out where and how to deploy our range of APs. Since we have relatively few resources we used a combination of what we had lying around, purchased, and donated harware. 

* 3x Cisco 1230
* 1x Cisco 1142
* 1x Cisco 1131
* 1x Cisco 1252

Looking at the map of the floor, I wanted to place the channel-bonded APs in the areas with the largest concentration of users (the Lounge and the User Center). The others would be dispersed across the other public rooms on floor.

[[ IMAGE HERE ]]

## Wireless Domain Services
Cisco WDS is essentially a poor-mans controller. WDS allows you to do authentication once across your wireless domain and do relatively seamless handoff between access points. I had an extra 1230 laying around without antennas so I parked it in my server room and configured it to act as the WDS master. When a client attempts to authenticate to an AP, the auth data is sent to the WDS server where it is then processed and a response sent to the AP to let them in or not. If the client roams to another AP then the WDS server promises the new AP that the client is OK and skips the authentication phase. 

This is the only device that will ever talk to the RADIUS server, so all of the configuration for that is only needed once. 

NOTE: In this example the WDS server is at IP address 192.168.0.250 and the RADIUS server is at 192.168.0.100.

```
aaa new-model
!
!
aaa group server radius rad_local
 server 192.168.0.250 auth-port 1812 acct-port 1813
!
aaa group server radius rad_eap
 server 192.168.0.100 auth-port 1812 acct-port 1813
!
aaa authentication login eap_local group rad_local
aaa authentication login eap_methods group rad_eap
!
radius-server local
  no authentication mac
  nas 192.168.0.250 key 7 SUPERSECRETKEYHERE 
  user wds-authman nthash 7 AUTHMANPASS 
!
radius-server host 192.168.0.250 auth-port 1812 acct-port 1813 key 7 SUPERSECRETRADIUSKEYHERE 
radius-server host 192.168.0.100 auth-port 1812 acct-port 1813 key 7 SUPERSECRETRADIUSOTHERKEYHERE 
!
!
wlccp ap username wds-authman password 7 AUTHMANPASS 
wlccp authentication-server infrastructure eap_local
wlccp authentication-server client eap eap_methods
  ssid prettyflyforawifi
wlccp authentication-server client leap eap_local
  ssid prettyflyforawifi
wlccp wds mode wds-only
wlccp wds priority 255 interface BVI1
```

The access points need to use the AP username defined above to talk to the WDS server. RADIUS configuration here should be optional.
```
aaa new-model
!
aaa group server radius rad_eap
 server 192.168.0.100 auth-port 1812 acct-port 1813
!
aaa authentication login default local
aaa authentication login eap_methods group rad_eap
dot11 ssid prettyflyforawifi 
   authentication open eap eap_methods
   authentication network-eap eap_methods
   authentication key-management wpa
   guest-mode
!
radius-server host 192.168.0.100 auth-port 1812 acct-port 1813 key 7 SUPERSECRETRADIUSKEY 
!
wlccp ap username wds-authman password 7 AUTHMANPASS 
wlccp ap wds ip address 192.168.0.250
```
