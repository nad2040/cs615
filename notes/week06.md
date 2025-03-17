# Simple Request ktrace

`tcpdump -w /tmp/simple.pcap port not 22 >/dev/null 2>&1 &`
`arp -d -a`
`ktrace -i telnet -K www.yahoo.com 80`
`HEAD / HTTP/1.0`

`ktrace` shows syscalls, namei translations, signal processing, and I/O
- basically the equivalent of strace and dtrace

`man nsswitch.conf`
`man hosts` - predates Domain Name System. takes precedence over DNS
`man resolv.conf` - specifies how DNS resolution should be done

What exactly happens?
- Local host connects to remote host.
  - turn remote hostname into IP address
    - determine how to resolve hostnames by via /etc/nsswitch.conf
    - look in /etc/hosts, then fail over to DNS
    - determine DNS server to use via /etc/resolv.conf
    - open socket to well-defined port on DNS server, send query, receive response
  - open socket to requested port on remote IP
- Local host sends data.
- Remote host replies with data.


The application made two separate requests to the DNS server. Why?

# Simple Request tcpdump

we see some packets from censys. they are scanning our ports.

arp packets - link-layer lookups of the MAC address

`route -n get default` show route

DNS uses UDP port 53

Quad A record: AAAA

TCP port 80:
SYN SYN-ACK ACK
push flag - data should be sent immediately

Remote sends FIN packet with FIN flag set
Host responds with ACK
Host sends FIN packet with FIN flag set
Remote responds with ACK
connection completed

HTTP and DNS are application layers
TCP and UDP are transport layers
IPv4 and IPv6 are network layers
ARP is link/physical layer

Inspect our tepdump output in detail. You should notice (at least) two things:
- We observe ARP requests from/to the default router before we talk to our DNS server. why is that?
- Look at the ARP replies from the DNS server and the default router. What can you
  deduce about the layer 2 network our instance is on from them?

# ARP and NDP

NDP (Neighbor Discovery Protocol) is IPv6 equivalent of ARP.

arp -a shows arp cache

even after a simple ping, there is lots of ARP traffic.

Gratuitous ARP - when there is a is-at response without a who-as request.

who-has - destination FF:FF:FF:FF:FF:FF - a broadcast request

is-at - reply is delivered via unicast. if the system wants to announce its location,
  it can send a gratuitous ARP, which is broadcast.

if multiple subnets are in the same VLAN, some IPs can be outside your subnet
but still in your broadcast domain

NDP utilizes ICMPv6
`sudo ndp -c` flushes ndp table
`ndp -n -a` shows table

`route -n get -inet6 default` gets default route for ipv6

ICMP neighbor solicitation
the low order 24 bits of the address is combined with the ff02::1:ff prefix
this is the solicited-node multicast address. neighbor solicitation sends a who-has request here.
the default route replies to global address. Using unicast tgt-is message
then default route asks who do i need to reply to.
then we reply with link local

sending an ICMP to ff02::1 (all-nodes multicast address) results in replies from all IPs in the multicast address.

the who-has request is multicast to every address matching the low order 24 bits.

NDP can also use the all-nodes multicast address to send a response to all
nodes in the link-local broadcast domain, similar to ARP.

Address Resolution Protocol (ARP): RFC826
- who-has broadcast (to FF:FF:FF:FF:FF:FF MAC)
- is-at unicast (unless gratuitous)

Neighbor Discovery Protocol (NDP): RFC4861
- neighbor solicitation ("who has") to Solicited-Node Multicast Address
- neighbor advertisement ("tgt is") to Global Unicast Address
- unsolicited neighbor advertisements to All-Nodes Multicast Address

# ICMP

Type: 0x04 on echo reply and 0x08 on echo request
Code: 0
Checksum: self-explanatory
Identifier: makes it easier to group packets together.
Sequence number: self-explanatory
Data: you can put arbitrary data in the packet... most places deny it

`sudo tcpdump -n -t -v "ip and (icmp or udp)"`
`traceroute -n -q 1 www.yahoo.com`

UDP to port 33435 with ttl 1
response ICMP time exceeded in transit -> so we learn the IP address.

since each hop decrements the ttl, when it hits 0, they must respond with the ICMP.

we trick the system to send a destination unreachable message by picking a random UDP port.

`sudo tcpdump -n -t -v "ip6 and (udp or (icmp6 && (ip6[40] == 3) || ip6[40] == 1))"`
`traceroute6 -q 1 -n www.yahoo.com`

hlim = hop limit

IPv4 PATH MTU discovery
- used to determine the max packet size that will make it to the destination without fragmentation
- works similarly to traceroute

What other ICMP types are out there?
- it will explain why it's bad to block ICMP across the board.

Some traceroute outputs *? why?

What is AS0 in the traceroute output?

---

# In Class

Arp poisoning can happen if you can get on the same layer 2 network

There is MACSec, but most layer 2 networks are unauthenticated.

# HW Review

Private networks do not get AS numbers.

- Anycasting - requests can be rerouted to a different location
- CDN deployment in different locations

DNS-based geolocation resolution

MaxMind: shows information about hops as well
https://www.maxmind.com/en/home

Some hop machines might choose not to respond, so you'll get a *.

Not all information is reported in geofeeds like
https://ipgeo.akamai.com/akamai-geofeed.csv

gps-coordinates.org

for the differences between shared, private, etc IP spaces
- https://en.wikipedia.org/wiki/IPv4#Special-use_addresses

100.64/10 is private shared address space.

This is why the AS was *...

https://bgp.tools/

what is an A or AAAA record?

traffic on our loopback interface might not be captured when we're
capturing on our main interface.

also systemd could ask another resolver. it is a layer
in between that caches DNS lookups.

systemd says local resolver looks at /etc/hosts

gai.conf
- can configure whether you do A and AAAA record queries
  and which result to prefer.

https://netmeister.org/blog/images/network-troubleshooting.png

