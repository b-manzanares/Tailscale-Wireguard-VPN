<h1>Tailscale-Wireguard-VPN</h1>

<h2>🎯 Objective</h2>

<p>Issue: I was scheduled to travel for work, but I needed a secure and simple way of remotely accessing resources on a home network. I did not want a VPN where I was required to open ports on my router or experience high latency.&nbsp;&nbsp;</p>

<p>The goal was to set up Tailscale between devices on my home network—a server, a workstation, and a laptop—for remote connections. I chose Tailscale because it creates a peer-to-peer mesh network where devices connect directly to each other, instead of a hub-and-spoke topology, which has the potential of bottlenecks, since all traffic is routing through a single source. I needed to access a server on my home network, in order to send a Wake-on-LAN(WOL) packet to a workstation that was running programs that I needed to access. Through the use of sub-net routers, exit-nodes, and iptables I was able to set up a secure way of accessing the available resources remotely. The best feature of Tailscale is that it is fast, secure, reliable, and has minimal management overhead.&nbsp;</p>

<h2>🗺️ Architecture</h2>

<ul>
  <li>Ubuntu server</li>
  <li>Ubuntu workstation</li>
  <li>Windows laptop</li>
</ul>

<h2>📖 Skills Learned</h2>

<ul>
  <li>Sub-net routers&nbsp;</li>
  <li>Exit-nodes&nbsp;</li>
  <li>Iptables/Nftables</li>
  <li>SSH</li>
</ul>

<h2>🗺️ Topology Diagram</h2>

<p>[Below is the structural layout of the simulated enterprise edge environment, showcasing dual multihomed wan/isp setup]</p>

<p>🚧🚧🚧</p>

<h2>☑️Configuration & Verification Snippets</h2>

<h3>1. Subnet routing</h3>

<p>In order to access a virtual machine on my workstation, I needed to allow subnet routing, since I couldn't install a tailscale client directly on the virtual machine. I enabled my home network subnet 192.0.2.0/24, in order for traffic from my home network to communicate with the machines on the tailnet. Without Source Network Address Translation (SNAT), the devices on the home network wouldn't be able to communicate with the devices on tailscale since the ip address of the tailscale network is 100.X.X.X a different network. The device on the home network will drop the packet or send it to it's default gateway potentially breaking the connection. SNAT transforms the source IP of the router from a tailscale IP to an IP of your home network, and also changes the private IP address of the home device behind the subnet router to the subnet routers IP.&nbsp;</p>

<p>In order to allow communication between the tailscale network and the home network I needed to set up ipv4 forwarding on the Linux node acting as a router.</p>

<pre><code>echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf</code></pre>

<p>Essentially you are displaying 'net.ipv4.ip_forward = 1' to standard output, and then using that output contents to create a file /etc/sysctl.d/99-tailscale.conf, so that ipv4 forwarding stays active upon reboots.&nbsp;</p>

<p>After, we must advertise the routes on the Linux system acting as the subnet router.&nbsp;</p>

<pre><code>sudo tailscale set --advertise-routes=192.0.2.0/24</code></pre>

<p>We must then approve these routes in the tailscale admin console.&nbsp;</p>

<p>Now we can communicate with devices on the home network, that are unable to install tailscale. We have created a safe, secure, and effective way of communicating with our remote devices without the need of opening ports on the home gateway, causing a security risk.&nbsp;</p>

<h3>🔥🧱 Iptables - Allowing traffic for tailscale</h3>

<p>If you have a default policy DROP in your Linux host-based firewall iptables forward chain, you will need to add the following rule to allow this node to act as a exit node.&nbsp;</p>

<pre><code>iptables -A FORWARD -j ts-forward</code></pre>

<p>What this rule essentially does is, when a packet arrives at the subnet router, or exit node, it is evaluated in the FORWARD chain since traffic is to be routed to this device before being sent out to either another device on the network or over the internet. If there is a DROP policy, it will be dropped since the default policy is DROP. Since we added this rule to our iptables it will forward this packet to the chain ts-forward, created by tailscale, and allow this packet, if the packet matches the tailscale forwarding rule.&nbsp;&nbsp;</p>

<h2>*** ☑️Troubleshooting ***</h2>

<h3>Tailscale - wasn't authenticating</h3>

<p>There was an issue where tailscale couldn't authenticate, and it would remain offline. It was unable to authenticate because the Linux machine had the incorrect time, which causes issues for the HTTPS/TLS handshake. In order to fix this, I needed to either disable the expiry or enable NTP on the Linux machine in order to correctly receive the correct time. So i used chronyd, but i ran into an issue with iptables. I had to allow the NTP(Network time protocol) port and another port for NTS(Network_Time_security).&nbsp;</p>

<p>I installed chrony - a free software program that keeps computer clocks accurate by synching with a NTP server. The machine wasn't able to connect to any NTP servers.&nbsp;</p>

<pre><code>apt install chrony</code></pre>

<p>The firewall was blocking the port needed for connections to the NTP(123) server. I enabled the NTP port in the etc/nftables.conf file under the output chain and I was able to connect and receive an accurate time, fixing the authentication issues.&nbsp;</p>

<pre><code>udp dport 123 accept


**Iptables config**
iptables -A OUTPUT -p udp -m udp --dport 123 -j ACCEPT
iptables -A OUTPUT -p tcp -m tcp --dport 4460 -m comment --comment "Allow port for (NTS)Network_Time_security" -j ACCEPT</code></pre>

<p>[Photos of a failing NTP server - and how fixing it fixed the issue]</p>

<h3>🌐 Internet connectivity issues when using an exit node - troubleshooting via tcpdump & journalctl</h3>

<p>If you have an exit node on your tailnet and you aren't able to get to the internet. Check to make sure ipv4 forwarding is enabled and that your firewalls forwarding chain is accepting traffic from the ts-forward chain.&nbsp;&nbsp;</p>

<p>I used Tcpdump on the exit node to see if packets were being sent from a remote device, since it is using SNAT the device should use it's own IP address(private) address 192.0.2.200 to send a packet to 203.0.113.1. When ts-forward isn't included in the forwarding section it is being blocked by the firewall, because all traffic is being blocked.&nbsp;</p>

<p>If there is no traffic being generated for that destination address &amp; you use a logging rule in your iptables, you can see that traffic is being logged before ultimately being dropped by iptables. If you are familiar with Cisco devices, a default policy DROP acts as a implicit deny in a ACL. It checks everything in the list before ultimately ending with an implicit deny that will drop everything unless you say otherwise. In this case a line was added to the iptables forward section which will catch the blocked packets just for logging purposes then drop the packets.&nbsp;&nbsp;</p>

<pre><code>example@example:~$ sudo iptables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "FORWARDING-DROP: "</code></pre>

<p>You can use journalctl to check what is being dropped. The forwarding chain is receiving packets from the remote computer, through the tailscale interface, but it is being dropped. Here you see traffic is being sent to the forwarding section of the iptables from the tailscale0 interface, but it is being picked up by the logging rule before then being dropped.&nbsp;</p>

<pre><code>example@example:~$ sudo journalctl -k -g "FORWARDING-DROP: " --since "1 hour ago" | grep DST=203.0.113.8
Sep 04 20:19:46 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40333 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=386
Sep 04 20:19:47 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40334 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=387
Sep 04 20:20:37 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40337 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=390
Sep 04 20:20:38 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40338 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=391</code></pre>

<p>Here is the tcpdump before allowing the rule. You can see that there is no packets being sent to 203.0.113.8 because it is being blocked by the forwarding rule.&nbsp;</p>

<pre><code>example@example:~$ sudo tcpdump -n -i enp128s31f6 icmp and src host 203.0.113.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp128s31f6, link-type EN10MB (Ethernet), snapshot length 262144 bytes
0 packets captured
0 packets received by filter
0 packets dropped by kernel</code></pre>

<p>Here is the tcpdump after the ts-forward chain has been added to the users forwarding chain. The correct IP(Private IP) is being used for the ICMP echo request now instead of the tailscale(100.X.X.X) IP address.</p>

<pre><code>example@example:~$ sudo tcpdump -n -i enp128s31f6 icmp and dst host 203.0.113.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp128s31f6, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:19:46.356013 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 386, length 40
20:19:47.374407 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 387, length 40
20:19:48.383585 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 388, length 40
20:19:49.374118 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 389, length 40</code></pre>

<hr>

<p>Next additions<br>
Add Access control lists so that only specific users are able to access the other nodes on the tailnet.&nbsp;</p>
