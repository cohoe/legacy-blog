---
layout: post
title: "High-Availability with Multipath Routing"
date: 2013-01-03 00:34
comments: true
sharing: true
categories: Guides
---
<h2>Multipath Routing</h2>
Multipath routing is when you have a router with multiple equal-cost paths to a single destination. Since each of these routes carries the same weight, the router will distribute traffic across all of them roughly equally. You can read more on this technology on <a href="https://en.wikipedia.org/wiki/Equal-cost_multi-path_routing">Wikipedia</a>. This is more about the implementation of such a topology. 

<h2>The Topology</h2>
In my setup, I have three clients on the 192.168.0.0/24 subnet with gateway 192.168.0.254. Likewise I have three servers on the 10.0.0.0/24 subnet with gateway 10.0.0.254. I want my clients to always be able to access a webserver at host 172.16.0.1/32. This is configured as a loopback address on each of the three servers. This is the multipath part of the project. Each of the servers has the same IP address on a loopback interface and will therefore answer on it. However since the shared router between these two networks has no idea the 172.16.0.1/32 network is available via the servers, my clients cannot access it. This is where the routing comes into play. I will be using an OSPF daemon on my servers to send LSA's (read up on OSPF if you are not familiar) to the router. 
<img src="https://www.grantcohoe.com/sites/default/files/styles/galleryformatter_slide/public/guides/multipath.PNG" alt="" title="" class="image-galleryformatter_slide" />
<h2>Server Configuration</h2>
<h3>Firewall</h3>
You need to allow OSPF traffic to enter your system. IPtables supports the OSPF transport so you only need a minimum of one rule:
<pre class="brush:plain">
iptables -I INPUT 1 -p ospf -j ACCEPT
</pre>
Save you firewall configuration after changing it.

<h3>Package Installation</h3>
You need to install Quagga, a suite of routing daemons that will be used to talk OSPF with the router. These packages are available in most operating systems. 

<h3>Interface Configuration</h3>
Configure your main interface statically as you normally would. You also need to setup a loopback interface for the highly-available host. Procedure for this varies by operating system. Since I am using Scientific Linux, I have a configuration file in <code>/etc/sysconfig/network-scripts/ifcfg-lo:1</code>:
<pre class="brush:plain">
DEVICE=lo:1
IPADDR=172.16.0.1
NETMASK=255.255.255.255
ONBOOT=yes
</pre>
Make sure you restart your network process (or your host) after you do this. I first thought the reason this entire thing wasn't working was a Linux-ism as my FreeBSD testing worked fine. I later discovered that it was because hot-adding an interface sometimes doesn't add all the data we need to the kernel route table. 

<h3>OSPF Configuration</h3>
Quagga's ospfd has a single configuration file that is very Cisco-like in syntax. It is located at <code>/etc/quagga/ospfd.conf</code>:
<pre class="brush:plain">
hostname server1
interface eth0
router ospf
	ospf router-id 10.0.0.1
	network 10.0.0.0/24 area 0
	network 172.16.0.1/32 area 0
log stdout
</pre>
ospfd has a partner program called zebra that manages the kernel route table entries learned via OSPF. While we may not necessarily want these, zebra must be running and configured in order for this to work. Edit your <code>/etc/quagga/zebra.conf</code>:
<pre class="brush:plain">
hostname server1
interface eth0
</pre>
After you are all set, start zebra and ospfd (and make sure they are set to come up on boot).

<h2>Router Configuration</h2>
I am using a virtualized Cisco 7200 router in my testing environment. You may have to modify your interfaces, but otherwise this configuration is generic across all IOS:
<pre class="brush:plain">
R1>enable
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#router ospf 1
R1(config-router)#network 10.0.0.0 0.0.0.255 area 0
R1(config-router)#end
</pre>
This enables OSPF on the 10.0.0.0/24 network and will start listening for LSAs. When it processes a new one, the route will be inserted into the routing table. After a while, you should see something like this:
<pre class="brush:plain">
R1#show ip route ospf
     172.16.0.0/32 is subnetted, 1 subnets
O       172.16.0.1 [110/11] via 10.0.0.3, 00:04:42, GigabitEthernet1/0
                   [110/11] via 10.0.0.2, 00:04:42, GigabitEthernet1/0
                   [110/11] via 10.0.0.1, 00:04:42, GigabitEthernet1/0
</pre>
This is multipath routing. The host 172.16.0.1/32 has three equal paths available to get to it. 

<h2>Testing</h2>
To verify that my clients were actually getting load-balanced, I threw up an Apache webserver on each server with a static index.html file containing the ID of the server (server 1, server 2, server 3, etc). This way I can curl the highly-available IP address and can see which host I am getting routed to. You should see your clients get distributed out across any combination of the three servers. 

Now lets simulate a host being taken offline for maintenance. Rather than having to fail over a bunch of services, all you need to do is stop the ospfd process on the server. After the OSPF dead timer expires, the route will be removed and no more traffic will be routed to it. That host is now available for downtime without affecting any of the others. The router just redistributes the traffic on the remaining routes. 

But lets say the host immediately dies. There is no notification to the router that the backend host is no longer responding. The only thing you can do at this point is wait for the OSPF dead timer to expire. You can configure this to be a shorter amount of time, thus making "failover" more transparent. 

<h2>Summary</h2>
Multipath routing is basically a poor-mans load balancer. It has no checks to see if the services are actually running and responding appropriately. It also does not behave very well with sessions or data that involves lots of writes. However for static web hosting or for caching nameservers (the use that sparked my interest in this technology), it works quite well.
