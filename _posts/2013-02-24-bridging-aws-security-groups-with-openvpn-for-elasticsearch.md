---
layout: post
title: "Bridging AWS security groups with OpenVPN, for ElasticSearch"
description: ""
category: code
tags: ["sysadmin", "elasticsearch", "openvpn"]
---
{% include JB/setup %}

<div class="intro" markdown="1">
{% excerpt %}
Recently we moved a bunch of infrastructure off EngineYard onto AWS. If you're not familiar, EngineYard is a platform which itself is hosted on AWS. We had our entire stack (including elasticsearch) hosted with them, but they didn't support elasticsearch and so we were effectively paying a 2-3x premium (!) for the convenience of hosting and a simple-ish setup.

The goal was to move the elasticsearch install onto our own raw AWS setup, thus saving about $1,500 per month (!) with no downtime. Here's how I did it, and You Can Too(tm). This requires you to know how to install and run Chef recipes on at least one new box. Trust me, it's faster to set up a hosted chef account with opscode, and get it working, than it is to figure out how to install OpenVPN with self-signed CA and certs.
{% endexcerpt %}

[Elasticsearch](http://elasticsearch.org) is a tricky customer because it has a discovery protocol where it communicates its public IP address with other nodes in the cluster, and you can't easily configure how this happens. I couldn't find any other documentation or posts about the topic, so I'm recording it for posterity here. It also is completely unsecure, which means, opening it up to the wider internet is Very Bad. It communicates to your app over http (port 9200) and with itself over TCP (9300), so we can't just use http-auth and SSL. So, openVPN.

This technique is usable any other time you want to bridge security groups where you don't own them all, but have ssh/root access to the individual servers. (AWS allows you to open up ports between your own groups)
</div>

<div class="tldr" markdown="1">
Overview
---

We're going to set up a routed-tunnel openVPN network, set up new elasticsearch servers outside the security group, then tweak elasticsearch to gossip/communicate with its other nodes with some IPtables routing tricks. Finally, we'll secure the new install with HAproxy.

</div>

OpenVPN
----

OpenVPN is fantastic, but complex. It creates a virtual network on any set of machines that can communicate with each other over TCP. It creates a new virtual network card (interface) with its own new IP address, so you can seamlessly communicate as though all machines were on a local network together. You can [read all about it](http://openvpn.net/index.php/open-source/documentation/howto.html) at their excellent site, but I've saved you most of the work here below.

The easiest way to install it is via the [Chef recipe](http://community.opscode.com/cookbooks/openvpn) - this recipe solves most of the issues you'll have generating certificate authorities and keys and so on. If you are a glutton for punishment, you will need to create a bunch of self-signed certs, and have a signing authority working. I tried to do this for a whole day and ended up with the chef recipe that does it magically.

So, first you set up the openVPN server. This box will be the conduit for all traffic. In our testing, a small instance was big enough for a cluster of three 2XL machines running at about 20% load. Let's assume you have your chef recipes uploaded and working with knife, and that we're using chef-solo and the hosted solution.

    [dev] $ knife ec2 server create -f m1.small -I ami-3d4ff254 -N openvpn --sudo

(be sure and apt-get update and apt-get upgrade before you do anything)

To install the openVPN server, you just set the node's run_list to recipe[openvpn].

    [dev] $ knife node run_list add openvpn 'recipe[openvpn],recipe[openvpn::users]'
  
Create a data bag called "users", so that it looks like this

    [dev] $ knife data bag create users elast1
  
With the contents consisting of only an `id`
  
    { id: "elast1" }
  
Create a users data bag for each of your 'old' servers (elast1, elast2, etc), and a bag for each of your new servers.

Now, run `chef-solo` on your openvpn server and it should install and create a bunch of .tgz files, one for each client machine, in `/etc/openvpn/keys`.

Make sure it all starts up OK (`sudo /etc/init.d/openvpn start` and `sudo ifconfig` should show a "tun" interface with IP of 10.8.0.0), and then we can install the clients.

Now edit the security group (in the AWS web app) and open port 1194 TCP for this machine.

OpenVPN Clients
-----

Now, on your 'old' machines, you want to set up openVPN clients. We don't need chef here; it's a one-liner, and these boxes are going to die soon anyway. (EngineYard uses gentoo, so we need emerge. You may need to unfuck their packages.mask)

    sudo emerge openvpn

Now, you want to edit /etc/openvpn/cilent.conf and set it to someting like this:

    client

    dev tun
    proto udp
    port 1194
    persist-tun
    persist-key
    keepalive 10 60

    remote ec2-12-34-56.compute-1.amazonaws.com 1194

    cert /etc/openvpn/keys/elast1.crt
    key  /etc/openvpn/keys/elast1.key
    ca   /etc/openvpn/keys/ca.crt

    comp-lzo

    tls-auth /etc/openvpn/keys/ta.key 1

..replacing the compute-1 address with that of your openvpn server. Copy the tgz of certificate files you just generated with the chef-solo run from the server and unzip them. You also need to copy ta.key from the server.

Now, start openvpn on the client, and it should magically work; check `sudo ifconfig` and see if you have an ip address (again, in the range 10.8.0.x). You should be able to ping 10.8.0.1. You should also be able to `curl http://10.8.0.4:9200` (replace with actual IP of another elasticsearch machine) and get results! Do this on each machine, and note their individual IP addresses for this new interface.

New boxes
------

Now, setup some new machines in your own AWS account with elasticsearch on them (no data yet). Install openVPN on them, the same as above. You should be able to curl the elasticsearch on the 'old' machines via their new 10.8.x IP addresses. 

Next we need to do some magic. Because Elasticsearch likes to communicate on startup and give out its own 'real' IP address, you want to trick the machines into communicating. The most evil way is with IPtables (note: you should read up on IPtables and how to save/load it on restart)

Make a list of every IP address: the amazon IP (looks something like 10.220.123.82) with its corresponding vpn IP (looks like 10.8.0.24). On each client machine, you want to run this set of commands for each pair other than the current box (as root).

    # iptables -t nat -A OUTPUT -d 10.220.123.72/32 -p tcp -m tcp --dport 9200 -j DNAT --to-destination 10.8.0.24:9200
    # iptables -t nat -A OUTPUT -d 10.220.123.72/32 -p tcp -m tcp --dport 9300 -j DNAT --to-destination 10.8.0.24:9300
    # iptables -t nat -A OUTPUT -d 10.220.123.72/32 -p udp -m udp --dport 9300 -j DNAT --to-destination 10.8.0.24:9300
    # ip route add 10.220.123.72/32 dev tun0


This tells IPtables to rewrite every packet going to 10.220.123.72, and actually send it to 10.8.0.24, on ports 9200 and 9300, and it further instructs all packets to 10.220.123.72 to go via the tun interface.

You should be able to test this again with `curl http://10.220.123.72:9200` on a machine in the 'new cluster'. That command won't work without iptables, but will work WITH iptables.

Turn it all on
-----

Watch the elasticsearch logs for errors! This is the stage where everything can break.

Edit the elasticsearch config on each of the 'old' machines. Beware, it appears that editing the files will dynamically reload the config. If you're using unicast, you want to change the addresses to match those of your openVPN network (10.8.0.4, 10.8.0.8, etc). Don't add the new IPs yet, just the 10.8x versions of the existing  cluster. Once that's working and you've verified that traffic is flowing over the VPN, modify your elasticsearch config again to increase the number of expected nodes.

Elasticsearch will magically re-allocate your data onto the new cluster. This might take hours for everything to go green again. In the meantime, you can set up HAProxy as a simple wrapper of SSL and basic auth on the 'new' cluster nodes so your app servers can gain access.

I used a fairly standard HAProxy install; ensure you have HTTP Basic Auth (user/pass, to authorize your app) as well as SSL (https, to encrypt communication over the wire). Test that you can access the new elasicsearch servers using their public address from your app servers. (We had to patch the gem we use for Elasticsearch to accept https and basic-auth; you may need to upgrade or patch it yourself also.)

So, before we pull the big switch o' doom, verify the following:

- cluster is green with old and new servers
- app servers can communicate with the new cluster via https and basic auth and the old cluster with straight http

Modify your app server config to point at only the new servers and restart them. Traffic should now be directly between app servers and new cluster over TCP and http, using the public IP addresses (not openVPN).

Decrease the number of expected nodes in your elasticsearch cluster by 1 and shut down one of the 'old' servers' elasticsearch process. Wait for it all to go green again. Repeat until there no more left running on the old cluster. Now you can shut down openVPN, terminate the openvpn server, and the old machines, and go get drunk.
