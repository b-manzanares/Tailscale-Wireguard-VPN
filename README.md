# Tailscale Setup

## 🎯 Objective

Issue: I was scheduled to travel for work and needed a secure, simple way to remotely access resources on my home network without opening router ports or experiencing high latency.

The goal was to set up Tailscale between devices on my home network—a server, a workstation, and a laptop for remote connection. I chose Tailscale because it creates a peer-to-peer mesh network where devices connect directly to each other, avoiding the bottlenecks of a hub-and-spoke topology where all traffic routes through a single source. I needed to access a server on my home network to send a Wake-on-LAN (WOL) packet to a workstation running required programs. Using subnet routers, exit nodes, and iptables, I established a secure method for remote resource access. Tailscale's standout features are its speed, security, reliability, and minimal management overhead.

## Architecture

* Ubuntu server
* Ubuntu workstation
* Windows laptop

## 📖 Skills Learned

* Subnet routers
* Exit nodes
* Iptables/Nftables
* SSH

## 🗺️ Topology Diagram

[Below is the structural layout of the simulated enterprise edge environment, showcasing dual multihomed WAN/ISP setup]

## ☑️ Configuration & Verification Snippets

### 1. Subnet Routing

To access a virtual machine on my workstation, I needed to configure subnet routing because I could not install the Tailscale client directly on the VM. I enabled my home network subnet (`192.0.2.0/24`) so traffic could flow between the home network and the tailnet. Without Source Network Address Translation (SNAT), home network devices cannot communicate with Tailscale devices because the Tailscale network (`100.x.x.x`) operates on a separate subnet. Otherwise, home network devices would drop the packet or send it to their default gateway, potentially breaking the connection. SNAT transforms the source IP of the router from a Tailscale IP to a home network IP, and changes the private IP address of the home device behind the subnet router to the subnet router's IP.

Alternatively, if your local router is configured with static routes back to the tailnet, you can disable SNAT:

```bash
tailscale set --snat-subnet-routes=false

```

To allow communication between the Tailscale network and the home network, I configured IPv4 forwarding on the Linux node acting as the router:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

```

This outputs `net.ipv4.ip_forward = 1` to standard output and appends it to `/etc/sysctl.d/99-tailscale.conf`, ensuring IPv4 forwarding persists across reboots.

Next, we must advertise the routes on the Linux system acting as the subnet router:

```bash
sudo tailscale set --advertise-routes=192.0.2.0/24

```

We must then approve these routes in the Tailscale admin console.

Now we can communicate with devices on the home network that are unable to install Tailscale. We have created a safe, secure, and effective way of communicating with our remote devices without the security risk of opening ports on the home gateway.

### Iptables - Allowing Traffic for Tailscale

If your Linux host-based firewall has a default `DROP` policy on the `FORWARD` chain, you will need to add the following rule to allow this node to act as an exit node:

```bash
iptables -A FORWARD -j ts-forward

```

When a packet arrives at the subnet router or exit node, it is evaluated in the `FORWARD` chain since traffic is routed through this device before being sent out to another network device or the internet. If there is a `DROP` policy, it will be discarded. Because we added this rule to our iptables, it forwards the packet to the `ts-forward` chain created by Tailscale and permits it if the packet matches Tailscale's forwarding rules.

## ☑️ Troubleshooting

### Tailscale Authentication Failures

Tailscale failed to authenticate and remained offline because the Linux machine had an incorrect system clock, which breaks HTTPS/TLS handshakes. To fix this, I needed to either disable certificate expiration or enable NTP on the Linux machine. I used `chrony`, but ran into an issue with iptables blocking the necessary ports.

I installed `chrony`, a free utility that keeps computer clocks accurate by synchronizing with an NTP server:

```bash
apt install chrony

```

The firewall was blocking the port needed for connections to the NTP (123) server. I enabled the NTP port in the `/etc/nftables.conf` file under the output chain, allowing me to connect, receive accurate time, and fix the authentication issues.

```
udp dport 123 accept

```

**Iptables config**

```bash
iptables -A OUTPUT -p udp -m udp --dport 123 -j ACCEPT
iptables -A OUTPUT -p tcp -m tcp --dport 4460 -m comment --comment "Allow port for (NTS) Network Time Security" -j ACCEPT

```

[Photos of a failing NTP server - and how fixing it resolved the issue]

### Internet Connectivity Issues When Using an Exit Node (Troubleshooting via tcpdump & journalctl)

If you have an exit node on your tailnet and cannot reach the internet, verify that IPv4 forwarding is enabled and that your firewall's forwarding chain accepts traffic from the `ts-forward` chain.

I used `tcpdump` on the exit node to check if packets were being sent from a remote device. Because SNAT is used, the device should use its private IP address (`192.0.2.200`) to send packets to `203.0.113.1`. When `ts-forward` isn't included in the forwarding section, traffic is blocked by the firewall's default policy.

If no traffic is generated for that destination address, you can use a logging rule in your iptables to see traffic logged before it is dropped. If you are familiar with Cisco devices, a default policy `DROP` acts as an implicit deny in an ACL, checking everything in the list before ending with a drop. In this case, a line was added to the iptables forward section to catch blocked packets for logging purposes:

```bash
example@example:~$ sudo iptables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "FORWARDING-DROP: "

```

You can use `journalctl` to check what is being dropped. Here, traffic arrives from the remote computer through the `tailscale0` interface, but gets caught by the logging rule and dropped:

```bash
example@example:~$ sudo journalctl -k -g "FORWARDING-DROP: " --since "1 hour ago" | grep DST=203.0.113.8
Sep 04 20:19:46 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40333 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=386
Sep 04 20:19:47 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40334 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=387
Sep 04 20:20:37 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40337 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=390
Sep 04 20:20:38 example kernel: FORWARDING-DROP: IN=tailscale0 OUT=enp128s31f6 MAC= SRC=100.X.X.X DST=203.0.113.8 LEN=60 TOS=0x00 PREC=0x00 TTL=127 ID=40338 PROTO=ICMP TYPE=8 CODE=0 ID=1 SEQ=391

```

Here is the `tcpdump` before allowing the rule, showing zero packets sent to `203.0.113.8` because they were blocked by the forwarding rule:

```bash
example@example:~$ sudo tcpdump -n -i enp128s31f6 icmp and src host 203.0.113.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp128s31f6, link-type EN10MB (Ethernet), snapshot length 262144 bytes
0 packets captured
0 packets received by filter
0 packets dropped by kernel

```

Here is the `tcpdump` after the `ts-forward` chain has been added. The correct private IP is now used for the ICMP echo request instead of the Tailscale (`100.X.X.X`) IP address:

```bash
example@example:~$ sudo tcpdump -n -i enp128s31f6 icmp and dst host 203.0.113.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp128s31f6, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:19:46.356013 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 386, length 40
20:19:47.374407 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 387, length 40
20:19:48.383585 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 388, length 40
20:19:49.374118 IP 192.0.2.200 > 203.0.113.8: ICMP echo request, id 1, seq 389, length 40

```

---

### Next Additions

Add Access Control Lists (ACLs) so that only specific users are able to access other nodes on the tailnet.


