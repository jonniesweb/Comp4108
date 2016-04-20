Lets do some arp spoofing, then figure out how to mitigate it!

Alright, what do I know about arp spoofing at the moment?

Arp stands for Address Resolution Protocol. It's an important bit of technology that maps ip addresses to mac addresses. It operates between the network and data link layers of the OSI model (layers 3 and 2, respectively).

Various attacks can be done using arp. One called arp poisoning involves sending out a malicious arp frame to the IP or mac address of the victim. This specially crafted arp frame contains the gateway's ip, but has the attackers mac address. When the victim receives this arp frame, they update their arp cache with the attackers mac address as the gateway. Now whenever the victim sends data to the gateway, it goes to the attackers computer. The attacker now is a man-in-the-middle for all data going in between the victim and the real gateway.

The attacker can monitor communications, perform session hijacking, Denial of Service, and modify traffic. Pretty powerful stuff.

Ettercap is a tool that specializes in doing arp poisoning attacks to perform a man-in-the-middle. The computer that runs ettercap must support promiscuous mode, aka monitor mode to inject the modified arp packets onto the network.

Since this attack works at the link layer and above, it should work wherever Ethernet frames are used. Both wired networks and wireless networks with or without security are vulnerable. There are some protections that can be done by administrators of the infrastructure. Client isolation is one method, where clients on the same network are blocked from communicating with each other. Another method is IP spoofing prevention, where the switch detects when packets destined from one IP should be actually going to another IP.

Alright, let's get this party started!

I'm on an open wifi right now, going 80km/h through the bush heading towards Toronto. I've got my laptop and phone with me, both connected to the same open wifi network. My plan of attack is to use ettercap on my laptop to send a spoofed arp packet to my phone, hopefully being able to setup a man-in-the-middle and monitor my traffic.

Let's do this! Launch the graphical interface for ettercap:

sudo ettercap -G

Great, now I specify ettercap to use my wlan0 interface, then enter the ip address of my phone, and the ip address of the gateway.

Then clicking on Mitm -> Arp Spoofing, I have begun the attack. Then Start -> Start Sniffing starts up the packet logging.

The first attempt I made I specified the IPs of the victim and the gateway in the wrong order. This sent the spoofed arp packet to the gateway instead of sending it to the victim. This little flub resulted in not capturing any data. After a little while of thinking "what did I do wrong?" I tried switching the order of the IPs and it worked; I was able to sniff the traffic.

Ettercap has a nice statistics screen showing you information about the traffic that is being snooped. It also allows you to save the useful packets and connection information into a file. I analysed this file using "etterlog -Ht tcp -f //80 out.ecp" and viewed the captured http packets from port 80.

Let's stop the monitoring and send out an arp frame to correct the man-in-the-middle. Doing this is made simple by going to Mitm -> Stop Mitm Attacks. Stopping the packet sniffing is done by clicking Start -> Stop Sniffing.

During the arp spoofing attack my phone's internet seemed much more latent. That's likely because my phone's internet connection bounced to the wifi access point, to my laptop, back to the wifi access point, then it goes to the internet. Normally the phone would communicate with the wifi access point, then go to the internet.

Analyzing the log files some more with etterlog I'm able to see all of the connections made during the capture session. I'm also took a look at some of the http packets, trying to see if I can find anything interesting.

Unfortunately there was a lot of garbled text. I'm thinking this must be encrypted, or etterlog/ettercap is formatting things incorrectly. Actually, now that I think of it, its probably compression done using gzip. Let me check some of the http headers and see if that's set.

Searching for "HTTP" brings me to the first occurrence of a http header. I search for "gzip" instead since I'm not finding much.

Aha! here's a http header with the gzip Content-Encoding header set. Right below it, where it's data is, is a bunch of garbled text. I think I'm right!

Well, if I'm serious about this packet snooping I could setup a filter to strip out the accepted Content-Encoding header from the http requests, or I could also do some post analysis by unzipping the gzipped data from the packets.

The best would be to take this ettercap packet capture and open it up in Wireshark, but I tried with no success.

After googling "ettercap wireshark" I found a guy saying that all you need to do is start the arp spoofing, don't capture packets using ettercap, but instead have wireshark monitor your interface.

Doing just that, I'm able to capture everything using wireshark. Wireshark is so nice and friendly to use. I'm glad that you're able to chain together these tools, benefiting from their strengths in the case of ettercap for spoofing arp and wireshark for offering the best damn packet filtering and analysis.

Wireshark is also great for saving images that go over http. I'm able to use the following filter to get all PNG images that are contained in http requests. "http.content_type=="image/png""

After wireshark returns the results I was able to right click on the PNG payload and export the bytes. Saving it with a .png extension then allowed me to view the image with an image viewer. So cool!

PS. This train runs a mysql server, and the seats are very uncomfortable.
