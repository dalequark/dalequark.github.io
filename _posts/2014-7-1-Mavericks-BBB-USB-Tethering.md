# BeagleBone Black Internet over USB in OSX Mavericks #

For quite some time, I was able to easily access the Internet from my BeagleBone
Black by enabling "Internet Sharing" in OSX Lion and attaching my BBB and MacBook
via an ethernet cable.  After upgrading to Mavericks 10.9.3, though, this setup 
completely broke.  After scouring the Internet and finding nothing to my remedy,
however, I did a little digging with tcpdump and found that a combination of 
port forwarding and nat settings on my MacBook needed to be changed (tldr; I needed to 
enable port forwarding, disable Internet Sharing, and change the settings in the 
OSX routing tool pfctl to enable a NAT).  Here's how
you can get your BeagleBone Black talking to the web over the provided USB mini cable:

1. Plug in the BBB and navigate to http://192.168.7.2 in Chrome/Firefox/what have you. 
The Linux Angstrom or Debian distribution (depending on which version of the BBB you've got) 
by default sets up a DHCP server, assigns itself the IP of 192.168.7.2, and hosts a static
"Getting Started" page at this address.  If this page doesn't load, try reinstalling the operating
system [here](http://beagleboard.org/getting-started#update).  In any case, by default you should
be able to ssh into the BBB with the username 'root':

	ssh root@192.168.7.2

2.  Once you can ssh into the 'bone, you need only to tell your MacBook to forward the BBB's
connection to the outside world (easier said than done).  In Terminal, enter:

	sudo sysctl -w net.inet.ip.forwarding=1

This instructs your MacBook to allow packets sent from the BBB over the local USB network to be
forwarded to the external network.  

3.  In Terminal on the Mac, run:
	
	ifconfig

Identify the name of the interface that a. your BBB is connected to and b. your WiFi/ethernet is running on.
For example, I'm connected to wifi on en0 because the ifconfig entry shows my IP address under "inet":

	en0: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
		ether 14:10:9f:e8:e1:62 
		inet6 fe80::1610:9fff:fee8:e162%en0 prefixlen 64 scopeid 0x4 
		inet 10.9.125.20 netmask 0xfffe0000 broadcast 10.9.255.255
		nd6 options=1<PERFORMNUD>
		media: autoselect
		status: active

The BBB is connecetd on en8, because it's IP address is 192.168.7.1 (and yours should be an interface with this 
inet address too):

	en8: flags=863<UP,BROADCAST,SMART,RUNNING,SIMPLEX> mtu 1486
		ether 7c:66:9d:6e:ff:58 
		inet 192.168.7.1 netmask 0xfffffffc broadcast 192.168.7.3
		media: autoselect
		status: active

4.  The problem I had was that my MacBook was not propertly translating the IP address of the BBB's packets.  In order
for the BBB to connect to the outside world, when it sends data to the Internet, the MacBook needs 
to change the IP address on the BBB's outgoing packets from 192.168.7.2 to it's own IP address (in my case, 10.9.125.20).
This is because 192.168.7.2 is a a *private* IP address that only the local USB network understands.  We need to enable 
a network address translator on the Mac (a NAT) in order to make the BBB's packets routable on the larger WiFi network.

Prior to OSX Lion, this would be done with ipfw/natd.  Now these tools have been outdated and replaced with pfctl.  We
change pfctl's config file to tell it to translate all packets from the BBB on en8 to an address managed by en0:

	sudo nano /etc/pf.conf

Add the line:

	nat on en0 from en8:network to any -> (en0)

where en0 is the interface connected to WiFi/ethernet, and en8 is the interface connected to the BBB,
underneath:

	scrub-anchor "com.apple/*"
	nat-anchor "com.apple/*"

Now run:

	sudo pfctl -f /etc/pf.conf 

If everything worked, the BBB should be connected.  From your SSH session with the BBB, test out the connection:

	ping 8.8.8.8

If you get an error like:

	network is unreachable

you may need to set the network gateway on the BBB:

	ip route add default via 192.168.7.1

If the BBB hangs on the ping, you may want to inspect its packets from your Mac.  On
OSX, run:

	tcpdump -vv -i en8

and try the ping again from the BBB.  Make sure there are incomping packets from 
192.168.7.2:

	192.168.7.2.http > 192.168.7.1.50223: Flags [.], cksum 0xbd2f (correct), seq 10, ack 19, win 243, options [nop,nop,TS val 1824893 ecr 455837959], length 0

If not, the error is likely on the side of the BBB.  If you see packets from the BBB from above but
ping still hangs, you might want to check that your NAT was actually enabled.  Run:

	tcpdump -vv -i en0 icmp

and run ping again from the BBB (also try to avoid using navigating to different web pages or making web requests).  
You should see:

	[an IP or address that is NOT 192.168.7.2] > google-public-dns-a.google.com: ICMP echo request, id 5921, seq 48, length 64
	11:02:25.640761 IP (tos 0x0, ttl 35, id 0, offset 0, flags [none], proto ICMP (1), length 84)
	    google-public-dns-a.google.com > [an IP or address that is NOT 192.168.7.2]: ICMP echo reply, id 5921, seq 48, length 64

If you see requests only from 192.168.7.2, your NAT is to blame.  Try restarting your computer and reloading pfctl.


Hope that helps!



