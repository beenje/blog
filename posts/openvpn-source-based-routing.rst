.. title: OpenVPN source based routing
.. slug: openvpn-source-based-routing
.. date: 2017-01-20 21:46:16 UTC+01:00
.. tags: pi,VPN,OpenWrt,AppleTV
.. category: linux
.. link: 
.. description: 
.. type: text

I already spoke about installing OpenVPN on a Raspberry Pi in another blog
`post </posts/installing-openvpn-on-a-raspberry-pi-with-ansible>`_.

I only connect to this VPN server to access content that requires a french IP address.
I use OpenVPN Connect App on my iPad and `Tunnelblick <https://tunnelblick.net>`_
on my mac.
It works nicely but how to use this VPN on my Apple TV 4?
There is no VPN client available...

End of last year I finally received my `Turris Omnia
<https://omnia.turris.cz/en/>`_ that I supported on Indiegogo.
It's a nice router running a free operating system based on
OpenWrt with automatic updates.
If you haven't heard about it, you should check it out.

Configuring OpenVPN client on OpenWrt
=====================================

Installing an OpenVPN client on OpenWrt is not very difficult.
Here is a quick summary.

1. Install `openvpn-openssl` package (via the
   webinterface or the command line)

2. I already have a custom client config that I generated with Ansible in
   this `post </posts/installing-openvpn-on-a-raspberry-pi-with-ansible>`_.
   To use this config, create the file `/etc/config/openvpn`::

    # cat /etc/config/openvpn
    package openvpn

    config openvpn myvpn
            # Set to 1 to enable this instance:
            option enabled 1
            # Include OpenVPN configuration
            option config /etc/openvpn/myclientconfig.ovpn

3. Add a new interface in `/etc/config/network`::

    config interface 'myvpn'
           option proto 'none'
           option ifname 'tun0'

4. Add a new zone to `/etc/config/firewall`::

    config zone
            option forward 'REJECT'
            option output 'ACCEPT'
            option name 'VPN_FW'
            option input 'REJECT'
            option masq '1'
            option network 'myvpn'
            option mtu_fix '1'

    config forwarding
            option dest 'VPN_FW'
            option src 'lan'

5. An easy way to configure DNS servers is to add fixed DNS for the WAN interface of the router.
   To use Google DNS, add the following two lines to the wan interface in `/etc/config/network`::

    # diff -u network.save network
    @@ -20,6 +20,8 @@
     config interface 'wan'
             option ifname 'eth1'
             option proto 'dhcp'
    +        option peerdns '0'
    +        option dns '8.8.8.8 8.8.4.4'

If you run `/etc/init.d/openvpn start` with this config, you should connect successfully!
All the traffic will go via the VPN. That's nice but it's not what I want.
I only want my Apple TV traffic to go via the VPN. How to achieve that?

Source based routing
====================

I quickly found this `wiki page
<https://wiki.openwrt.org/doc/networking/routing>`_ to implement source
based routing. Exactly what I want. What took me some time to realize is
that before to do that I had to ignore the routes pushed by the server.

With my configuration, when the client connects, the server pushes some
routes among which a default route that makes all the traffic go via the
VPN::

    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    default         10.8.0.21       128.0.0.0       UG    0      0        0 tun0
    ...

Ignoring the routes pushed by the server can be done with the `--route-noexec` option.
I tried to add `option route_noexec 1` to my `/etc/config/openvpn` file
but it had no effect. It looks like that when using a custom config, you
can't add other options there. You have to set everything in the custom
config. I added `route-noexec` to  my `/etc/openvpn/myclientconfig.ovpn` file and it worked!
No more route added. No traffic sent via the VPN.

We can now apply the changes described in the `Routing wiki page
<https://wiki.openwrt.org/doc/networking/routing>`_.

1. Install the `ip` package

2. Add the `10 vpn` line to `/etc/iproute2/rt_tables` so that it looks like
   this::

	# cat /etc/iproute2/rt_tables
	#
	# reserved values
	#
	255  local
	254  main
	253  default
	10   vpn
	0    unspec
	#
	# local
	#
	#1  inr.ruhep

3. We now need to add a new rule and route when starting the client.
   We can do so using the openvpn `up` command. Create the `/etc/openvpn/upvpn` script::

	# cat /etc/openvpn/upvpn
	#!/bin/sh

	client=192.168.75.20

	tun_dev=$1
	tun_mtu=$2
	link_mtu=$3
	ifconfig_local_ip=$4
	ifconfig_remote_ip=$5

	echo "Routing client $client traffic through VPN"
	ip rule add from $client priority 10 table vpn
	ip route add $client dev $tun_dev table vpn
	ip route add default via $ifconfig_remote_ip dev $tun_dev table vpn
	ip route flush cache

4. Create the `/etc/openvpn/downvpn` script to properly remove the rule and route::

	# cat /etc/openvpn/downvpn
	#!/bin/sh

	client=192.168.75.20

	tun_dev=$1
	tun_mtu=$2
	link_mtu=$3
	ifconfig_local_ip=$4
	ifconfig_remote_ip=$5

	echo "Delete client $client traffic routing through VPN"
	ip rule del from $client priority 10 table vpn
	ip route del $client dev $tun_dev table vpn
	ip route del default via $ifconfig_remote_ip dev $tun_dev table vpn
	ip route flush cache

5. We now have to add those scripts to the client config.
   Here is everything I added to my `/etc/openvpn/myclientconfig.ovpn` file::

    # Don't add or remove routes automatically
    # Source based routing for specific client added in up script
    route-noexec
    # script-security 2 needed to run up and down scripts
    script-security 2
    # Script to run after successful TUN/TAP device open
    up /etc/openvpn/upvpn
    # Call down script before to close TUN to properly remove the routing
    down-pre
    down /etc/openvpn/downvpn

Notice that the machine IP address that we want to route via the VPN is
hard-coded in the the upvpn and downvpn scripts.
This IP shall be fixed. You can easily do that by associating it to
the required MAC address in the DHCP settings.

The tunnel remote IP is automatically passed in parameter to the up and
down scripts by openvpn.

If we run `/etc/init.d/openvpn start` with this config, only the traffic
from the 192.168.75.20 IP address will go via the VPN!

Run `/etc/init.d/openvpn stop` to close the tunnel.

Conclusion
==========

This is a nice way to route traffic through a VPN based on the source IP
address.

You can of course use the router webinterface to stop and start openvpn.
In another post, I'll talk about an even more user friendly way to control
it.
