Jonathan Simpson
100898116

Total time: ~4 hours

January 15, 2016 7:40pm

Why is my wifi breaking up every so often. Pages hang when they load, but then they're good for a while.

5 minutes passes.

This is getting annoying. Lets apt-get wireshark and see if I can figure anything out.

Installed it, ran "wireshark" in my terminal. Oh yeah, I probably need root permissions to read all of the data. Exit that wireshark instance, this time entering "sudo wireshark" in my terminal.

Read the warning dialogs about how it may be dangerous to run Wireshark under root and how you can do it as a normal user. Spent a minute googling for it, but then gave up since I don't think it'll be an issue running as root.

Browse the web, waiting for the wifi to break up and the pages to hang while loading.

Alright, its happening now. Hop over to wireshark and start capturing everything on the wlan0 interface. It's been a while since I've used Wireshark, there's all these colours. There are packets coming in that are highlighted red. This may be a good thing to investigate.

Let it run for a few minutes, waiting for the wifi connection to go back to normal now and the web pages to load.

Looks like the web pages are loading again. I'm stopping the packet capture.

I'll quickly skim through the list of captured packets. I did a quick "ip a" to find out my wlan0 ipv4 address, to see what packets are to/from me. 192.168.1.5 to be exact.

What's all these consecutive Ethernet packets with the same data? Here is one:
1456	83.341566000	Cisco-Li_db:36:85	24:df:6a:4c:1d:1a	Ethernet	1538	Ethernet Unknown: Invalid length/type: 0x05f4 (1524)

I see that it's source is "Cisco-Li_db:36:85", which might be the Cisco Linksys wifi router. Let's verify this by logging into the web interface of the wifi router and finding out what it's mac addresses are. The router is flashed with DD-WRT, relieving a lot of the frustrations brought with the crappy OEM router firmware. Looks like it belongs to the "LAN MAC" interface of the router.

I then need to find out what the destination is. It's MAC is 24:df:6a:4c:1d:1a. My first guess is to look at the DHCP clients list in the web interface of the router. The list contains the MAC address. It refers to the IP 192.168.1.104, with a host name of android-4b0905fa3855190c. This seems like it's someone's Android phone.

I look at my Android phone and find what it's IP address is. 192.168.1.104. It matches. But I just got this phone  brand new the other day. Maybe it's because I rooted it? or could it be because the Nexus 6P is a new phone and Marshmallow (the new version of Android) is new and has bugs? I google for "ethernet unknown invalid length/type nexus 6p", try a few links and get nothing. Then I try "ethernet unknown invalid length/type android" and find some guy talking about his Galaxy Note doing the same thing, but his phone's CPU was at 100%. It's an old Android bug report from 2012. I don't think this is helpful.

I don't think this is a big issue since I don't notice anything wrong with my phone's wifi connection or its performance.

Anyways, back to what I was originally going to do, figure out why the wifi is spotty. Let's see what some of those red highlighted packets are up to.

Starting from the beginning there looks to be HTTP packets and TCP packets going on. The HTTP packets don't show anything since they look like they just have data, no headers or anything. The TCP packets are interesting though.

Here is one:
42	3.326454000	192.168.1.5	74.125.69.95	TCP	78	[TCP Window Update] 35790 > http [ACK] Seq=1 Ack=4294950281 Win=672 Len=0 TSval=40232826 TSecr=3250487297 SLE=1 SRE=2837
This computer is sending a ACK (TCP Window Update) packet to a remote server. Later on in the logs there is a TCP retransmission message. There is actually a lot of TCP retransition messages. They appear as black with orange text. I guess that's to make it really stand out.

The TCP retransmission occurs when we don't get an ACK for the data we've sent in a timely fashion. TCP does this since it assumes that the packet(s) didn't get delivered, so it sends the packet(s) again in an attempt to get the destination to ACK the packets we've sent to them.

Scrolling through even further I find a red TCP RST for the same remote IP. Its from this computer, meaning that we're closing the TCP connection because a timeout was reached and out TCP/IP stack assumes that something has disallowed the connection to exist.

I then try and filter out all of the other unrelated packets. I use Wireshark's filtering ability to only show packets from the ip 74.125.69.95, using the expression "ip.addr == 74.125.69.95". Much nicer.

Time to take a moment and appreciate the trip-hop I'm listening to. God-damn wifi! I'm trying to search SoundCloud on my laptop, but the wifi is breaking up again! Here's the mix: https://soundcloud.com/marshmellomusic/diploandfriends

Back to Wireshark, there appears to be TCP retransmissions of SYN-ACK messages from the remote server to this computer. That's interesting. It makes me think the wifi router is able to send messages back to me, so maybe the issue is somewhere inbetween my wifi router and the remote server?

Lets look at the Frame and maybe other packet headers for some timing information. Maybe we can find that packets are getting buffered somehwere.

Scratch that, wow, look at the "Win" fields of these two packets. They are the TCP window sizes, in bytes, of how much data the sender or receiver can take. I can't remember exactly which.

#1
73	5.661792000	192.168.1.5	74.125.69.95	TCP	78	[TCP Window Update] 35790 > http [ACK] Seq=1 Ack=4294950281 Win=738 Len=0 TSval=40233410 TSecr=3250487297 SLE=1 SRE=7091

#2
1641	87.447900000	192.168.1.5	74.125.69.95	TCP	66	43625 > http [ACK] Seq=439 Ack=38287 Win=105856 Len=0 TSval=40253857 TSecr=3233661660

That's a big difference. The window size being so small in #1 definitely shows that the TCP congestion control is at work trying to not flood the network with too much data. #2 is from later on when the wifi went back to normal.

At the beginning of the packet capture the TCP connection was already established. The remote server is sending us small amounts of data and we send back ACKs with the small window size. Looking at the Wireshark analysis, I see that the round trip time (RTT) of a TCP ack from us to the remote server back to us is 60 milliseconds. That seems normal.

I see an ICMP reply stating that a DNS query to 8.8.8.8, google's free DNS servers, is unable to reach the server since the port is unreachable. Weird, could google have something to do with this?

Time to benchmark my DNS. I found namebench, a program to find the fastest dns. Downloaded the source and now I'm running the python command line app in a terminal. It's running and I'm wondering if the wifi is crappy or not right now. I run a simple "ping google.com" as the test is going on.

Wow, all of these ICMP pings are in the seconds, that's bad.

Oh, maybe it was the 40 threads that the DNS benchmarking program was using to query dns servers that was slowing things down. The benchmark tool changed to 6 threads running the DNS queries and the ICMP ping time went back to sub 100 millisecond times.

Where did the time go?

It's almost 10pm and I haven't had dinner yet. Time to drop this and go eat.

The DNS benchmark finished. I'm going to give it a go and change my current DNS to the recommended ones. I'm pretty sure this won't help, but at least my DNS will be faster.

After configuring the DNS in the Xubuntu network-manager app, I wanted to check that it was updated in an actual configuration file. /etc/resolv.conf didn't contain it. Something called resolvconf didn't have anything in the manpages or its /etc/resolvconf files. Time to consult google. A quick search later shows that the "nmcli dev list" displays the current dns configuration. It still shows 8.8.8.8, maybe disconnecting and reconnecting the wifi would reset everything.

After disconnecting and reconnecting, the nmcli command then displayed the three DNS servers that I configured. Bingo.

Time to leave this for now and see if there are still any issues, and because it's 11pm already.



January 17, 2016 5:30pm

Conclusion

It was suggested by another student in the class over Discord that I should consider configuring RTS/CTS if my physical area is congested with wifi interference. I don't really think it's an issue since there's only < 10 other access points around me. Living beside the 417 and having large hunks of metal wizzing by day and night may have an effect, but I'm no physisist.

After asking my roommates about their wifi issues, it seems that they don't face the same problems as I do since their computers are connected via Ethernet cables, and they say that they don't have the issues I'm having. This concludes that it's a Wifi issue.

I have the hardware to try using another wireless router instead of using the current one. I will be giving that a try and seeing if it works any better.

My Wireshark skills were pretty rusty, but it was helpful to see what the actual TCP conenctions were doing during when the wifi was breaking up.