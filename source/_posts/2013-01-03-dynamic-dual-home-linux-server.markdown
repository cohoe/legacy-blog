---
layout: post
title: "Dynamic Dual-Home Linux Server"
date: 2013-01-03 01:15
comments: true
sharing: true
categories: Projects
alias: /projects/dual-home-linux
---
# Requirements
I have a shiny new VM server sitting in my dorm room. I have access to two networks, one operated by my dorm organization and the other provided to me by RIT. Both get me to the internet, but do so through different paths/SLAs. I want my server to be accessible from both networks. I also want to be able to attach VM hosts to NAT'd networks behind each respective network. This gives me a total of four possible VM networks (primary-external, primary-internal, secondary-external, secondary-internal). I have a Hurricane Electric IPv6 tunnel endpoint configured on the server, and want IPv6 connectivity available on ALL networks regardless of being external or internal. And to complicate matters, my external IP addresses are given to me via DHCP so I cannot set anything statically. Make it so.
# Dependencies
* IPTables
* EBTables
* Kernel 2.6 or newer
* IPCalc
You also need access to two networks.

# Script Setup
First, like any good shell script, we should define our command paths:
```
SERVICE=/sbin/service
IPTABLES=/sbin/iptables
IP6TABLES=/sbin/ip6tables
EBTABLES=/sbin/ebtables
IPCALC=/bin/ipcalc
```

Next we'll define the interface names that we are going to use. These should already be pre-configured and have their appropriate IP information.
```
PRIMARY_EXTERNAL_INTERFACE="primary-net"
PRIMARY_INTERNAL_INTERFACE="primary-nat"
SECONDARY_EXTERNAL_INTERFACE="secondary-net"
SECONDARY_INTERNAL_INTERFACE="secondary-nat"
IPV6_TUNNEL_INTERFACE="he-ipv6"
```

# Subnet Information
Now we need to load in the relevant subnet layouts. In an ideal world this would be set statically. However due to the nature of the environment I am in, both of my networks serve me my address via DHCP and is subject to change. This is very dirty and not very efficient, but it works:
```
# Here we will get the subnet information for the primary network
PRIMARY_EXTERNAL_NETWORK=`$IPCALC -n $(ip -4 addr show dev $PRIMARY_EXTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6) | cut -d = -f 2`
PRIMARY_EXTERNAL_PREFIX=`$IPCALC -p $(ip -4 addr show dev $PRIMARY_EXTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6) | cut -d = -f 2`
PRIMARY_EXTERNAL_NETWORK="$PRIMARY_EXTERNAL_NETWORK/$PRIMARY_EXTERNAL_PREFIX"
# Then the NAT'd network behind the primary
PRIMARY_INTERNAL_NETWORK=`ip -4 addr show dev $PRIMARY_INTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6 | sed "s/[0-9]\+\//0\//"`
# Now the subnet information for the secondary network
SECONDARY_EXTERNAL_NETWORK=`$IPCALC -n $(ip -4 addr show dev $SECONDARY_EXTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6) | cut -d = -f 2`
SECONDARY_EXTERNAL_PREFIX=`$IPCALC -p $(ip -4 addr show dev $SECONDARY_EXTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6) | cut -d = -f 2`
SECONDARY_EXTERNAL_NETWORK="$SECONDARY_EXTERNAL_NETWORK/$SECONDARY_EXTERNAL_PREFIX"
# And it's NAT'd network.
SECONDARY_INTERNAL_NETWORK=`ip -4 addr show dev $SECONDARY_INTERNAL_INTERFACE | grep "inet" | cut -d ' ' -f 6 | sed "s/[0-9]\+\//0\//"`

# This is where we load in the IP addresses of the interfaces
PRIMARY_EXTERNAL_IP=`ip -4 addr show dev $PRIMARY_EXTERNAL_INTERFACE | grep inet | cut -d ' ' -f 6 | sed "s/\/[0-9]\+$//"`
PRIMARY_INTERNAL_IP=`ip -4 addr show dev $PRIMARY_INTERNAL_INTERFACE | grep inet | cut -d ' ' -f 6 | sed "s/\/[0-9]\+$//"`
SECONDARY_EXTERNAL_IP=`ip -4 addr show dev $SECONDARY_EXTERNAL_INTERFACE | grep inet | cut -d ' ' -f 6 | sed "s/\/[0-9]\+$//"`
SECONDARY_INTERNAL_IP=`ip -4 addr show dev $SECONDARY_INTERNAL_INTERFACE | grep inet | cut -d ' ' -f 6 | sed "s/\/[0-9]\+$//"`

# We get the gateways by pinging once out of the appropriate interface to the multicast address of All Routers on the network. 
PRIMARY_GATEWAY_IP=`ping -I $PRIMARY_EXTERNAL_IP 224.0.0.2 -c 1 | grep "icmp_seq" | cut -d : -f 1 | awk '{print $4}'`
SECONDARY_GATEWAY_IP=`ping -I $SECONDARY_EXTERNAL_IP 224.0.0.2 -c 1 | grep "icmp_seq" | cut -d : -f 1 | awk '{print $4}'`
```

# System Configuration
In order to get most of these features working, we need to enable IP forwarding in the kernel.
```
echo "Enabling IP forwarding..."
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1
```

# Routes
The routes are the most critical part of making this whole system work. We use two cool features of the linux networking stack, Route Tables and Route Rules. Route Tables are similar to your VRF tables in Cisco-land. Basically you maintain several different routing tables on a single system in addition to the main one. In order to specify what table a packet should use, you configure Route Rules. A rule basically says "Packets that match this rule should use table A". In this system I use rules to force a packet to use a route table based on its source address.
```

# Kernel IP Routes
echo "Setting system default gateways"
ip route del default
ip route add default via $PRIMARY_GATEWAY_IP
ip -6 route add ::/0 dev $IPV6_TUNNEL_INTERFACE

echo "Adding default routes for primary external network..."
ip route flush table $PRIMARY_EXTERNAL_INTERFACE-routes
ip route add $PRIMARY_EXTERNAL_NETWORK dev $PRIMARY_EXTERNAL_INTERFACE src $PRIMARY_EXTERNAL_IP table $PRIMARY_EXTERNAL_INTERFACE-routes
ip route add default via $PRIMARY_GATEWAY_IP table $PRIMARY_EXTERNAL_INTERFACE-routes

echo "Adding default routes for secondary external network..."
ip route flush table $SECONDARY_EXTERNAL_INTERFACE-routes
ip route add $SECONDARY_EXTERNAL_NETWORK dev $SECONDARY_EXTERNAL_INTERFACE src $SECONDARY_EXTERNAL_IP table $SECONDARY_EXTERNAL_INTERFACE-routes
ip route add default via $SECONDARY_GATEWAY_IP table $SECONDARY_EXTERNAL_INTERFACE-routes

echo "Creating routes for primary internal network..."
ip route flush table $PRIMARY_INTERNAL_INTERFACE-routes
ip route add $PRIMARY_INTERNAL_NETWORK dev $PRIMARY_INTERNAL_INTERFACE table $PRIMARY_INTERNAL_INTERFACE-routes
ip route add default via $PRIMARY_GATEWAY_IP table $PRIMARY_INTERNAL_INTERFACE-routes

echo "Creating routes for secondary internal network..."
ip route flush table $SECONDARY_INTERNAL_INTERFACE-routes
ip route add $SECONDARY_INTERNAL_NETWORK dev $SECONDARY_INTERNAL_INTERFACE table $SECONDARY_INTERNAL_INTERFACE-routes
ip route add default via $SECONDARY_GATEWAY_IP table $SECONDARY_INTERNAL_INTERFACE-routes

echo "Creating route rules for primary external network..."
ip rule del from $PRIMARY_EXTERNAL_IP
ip rule add from $PRIMARY_EXTERNAL_IP table $PRIMARY_EXTERNAL_INTERFACE-routes

echo "Creating route rules for secondary external network..."
ip rule del from $SECONDARY_EXTERNAL_IP
ip rule add from $SECONDARY_EXTERNAL_IP table $SECONDARY_EXTERNAL_INTERFACE-routes

echo "Creating route rules for primary internal network..."
ip rule del from $PRIMARY_INTERNAL_NETWORK
ip rule add from $PRIMARY_INTERNAL_NETWORK lookup $PRIMARY_INTERNAL_INTERFACE-routes

echo "Creating route rules for secondary internal network..."
ip rule del from $SECONDARY_INTERNAL_NETWORK
ip rule add from $SECONDARY_INTERNAL_NETWORK lookup $SECONDARY_INTERNAL_INTERFACE-routes
```

# Firewall
My script also maintains the host firewall rules, so we'll enable those:
```
echo "Reseting firewall rules..."
$SERVICE iptables restart
$SERVICE ip6tables restart

echo "Setting up IPv4 firewall rules..."
$IPTABLES -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
$IPTABLES -A INPUT -p ipv6 -j ACCEPT
$IPTABLES -A INPUT -p icmp -j ACCEPT
$IPTABLES -A INPUT -i lo -j ACCEPT
$IPTABLES -A INPUT -s mason.csh.rit.edu -p tcp --dport 5666 -j ACCEPT
$IPTABLES -A INPUT -p tcp --dport 53 -j ACCEPT
$IPTABLES -A INPUT -p udp --dport 53 -j ACCEPT
$IPTABLES -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
$IPTABLES -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
$IPTABLES -A INPUT -j REJECT --reject-with icmp-host-prohibited

echo "Setting up IPv6 firewall rules..."
$IP6TABLES -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
$IP6TABLES -A INPUT -p ipv6-icmp -j ACCEPT
$IP6TABLES -A INPUT -i lo -j ACCEPT
$IP6TABLES -A INPUT -p tcp  --dport 53 -j ACCEPT
$IP6TABLES -A INPUT -p udp  --dport 53 -j ACCEPT
$IP6TABLES -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
$IP6TABLES -A INPUT -j REJECT --reject-with icmp6-adm-prohibited

echo "Setting up firewall rules for primary internal network..."
$IPTABLES -A FORWARD -s $PRIMARY_INTERNAL_NETWORK -j ACCEPT
$IPTABLES -t nat -A POSTROUTING -s $PRIMARY_INTERNAL_NETWORK -j SNAT --to-source $PRIMARY_EXTERNAL_IP

echo "Setting up firewall rules for secondary internal network..."
$IPTABLES -A FORWARD -s $SECONDARY_INTERNAL_NETWORK -j ACCEPT
$IPTABLES -t nat -A POSTROUTING -s $SECONDARY_INTERNAL_NETWORK -j SNAT --to-source $SECONDARY_EXTERNAL_IP

echo "Setting up default firewall actions..."
$IPTABLES -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
$IPTABLES -A FORWARD -j REJECT --reject-with icmp-port-unreachable
```

# Router Advertisements
One of the features of my networking setup is that all networks, no matter internal or external have IPv6 connectivity. This is achieved by sending Router Advertisements out all interfaces. This is all well and good for the VMs that I am hosting, but these advertisements travel out of the physical interfaces to other hosts on the network! This is not good in that I am allowing other users to use my IPv6 tunnel interface. This puzzled me for a long time until I discovered a handy little program called "ebtables". Ebtables is basically iptables for layer 2. As such, I was able to filter all router advertisement broadcasts out of the physical interfaces.
```
echo "Blocking IPv6 router advertisement to the world..."
$SERVICE radvd stop
$SERVICE ebtables restart
$EBTABLES -A OUTPUT -d 33:33:0:0:0:1 -o eth1 -j DROP
$EBTABLES -A OUTPUT -d 33:33:0:0:0:1 -o eth0 -j DROP
$SERVICE radvd start
```

# End Result
You now have a dual-homed server with two external networks, two internal networks, and IPv6 connectivity to all. Both external networks can receive their configuration via DHCP and will dynamically adjust their routing accordingly. 
<img src="https://www.grantcohoe.com/sites/default/files/enterprise.PNG" width="662" height="457" alt="" title="" />
