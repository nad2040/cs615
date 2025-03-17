# The Domain Name System

ARPANET - the early days

manual page for hosts
- we used to have one central hosts file that was shared around.
- RFC 597 - had old hosts list
- RFC 810 - successor to that list. Elizabeth Feinler
  - defined DoD host table
  - NET/GATEWAY/HOST : IP : ...
  - the internet used to have 1305 hosts

early on, addresses were assigned manually by calling up the
Network Information Center (NIC) at the Stanford Research Institute (SRI)

Feinler and Jon Postel handled the internet.
Feinler came up with the concept of using domains

Paul Mockapetris proposed a DNS in RFC 882 and 883 in 1983

Grad students at Berkeley were the first to implement it:
Berkeley Internet Name Domain (BIND) 1984
- it still is one of the biggest name servers, but now is under control of the Internet Systems Consortium (ISC)

Root of the DNS tree is "."

Then there are Top-Level Domains (TLDs):
- arpa
- com
- edu
- org
- net
- gov
- mil

Then there are second-level domains and third-level domains

At each level, you can hand off control over a subdomain to a different entity.

Every node must have a label as well as 0 or more Resource Records (RRs)

Most common Resource Records are the IP address associations via the A and AAAA records.
There's also the CNAME (canonical name) record - kinda works like symlinks.
MX records define the mail servers responsible for a domain.
NS records define the name servers responsible for a given zone or rocords associated with DNSSEC.

Fully Qualified Domain Name (FQDN) requires the last . to be present
- you may be able to avoid a paywall by adding that trailing dot...

RFC 920 defined a set of general purpose domains
- com (commercial use)
- edu (educational use)
- gov (government use)
- mil (military use)
- org (any organization, nowadays non-profits)
- arpa (temporarily used for arpanet administrative purposes)
- country-code TLDs (ccTLDs), using ISO 3166 code
  - de, fr, ar, ... there are internationalized codes for egypt, hong kong, and europe with cyrillic and other alphabets.
- internationalized generic TLDs (IDN gTLD)
- sponsored TLDs (sTLD) - used for smaller/narrow groups/communities
  - cat (catalan), jobs (human resource managers), xxx (voluntary option for porn websites)
- "new" generic TLDs
  - auto, blue, mom, top, video

How many TLDs in total?
`curl -s ftp://rs.internic.net/domain/root.zone`
`| grep "IN\tNS\t" | awk '{print $1}' | sort -u | wc -l`
1504 as of March 2021
1444 as of March 2025

Managing Domains
- IANA manages root zone (.), and (arpa.) and (int.)
- all other domains are managed by domain name registries
- registries may outsource the registration of domain names to registrars or be their own registrar.
- registries control the policy of allocation
  - e.g. you must have your page in catalan if you want to use .cat
- registries can (and do) censor, revoke, change, ... entries

If you control one node, you control the entire subtree.
Keep an eye on your NS records.

# How Name Resolution Works

```sh
sudo tcpdump -w /tmp/dns.pcap port 53 >/dev/null 2>&1 &
nslookup www.yahoo.com
fg
^C
```

DNS uses port 53

nslookup uses the nameserver in /etc/resolv.conf

result is marked as non-authoritative. this is because the nameserver that
provided the answer is not in charge of the zone

Query for an A record.
Response in next packet.
5 answer resource records.
- canonical name for www.yahoo.com
- different ip addrs that the CNAME resolves to.

Query for AAAA record.
Responses are for those of the CNAME
Flags show that the response was non-authoritative

Authoritative servers provide authoritative answers;
a resolver relays the answers it gets after asking the right authoritative servers.
Often, these resolvers cache recent results, so they are often called caching resolvers.

A simple request can lead to multiple queries and multiple record RR types.
- CNAME record for resolving the real domain of yahoo
- several A records (IPv4 addresses)
- several AAAA records (IPv6 addresses)

## Okay, now lets get an authoritative response:

```sh
sudo tcpdump -w /tmp/dns.pcap port 53 >/dev/null 2>&1 &
nslookup -query=ns www.yahoo.com
```

authoritative answers can be found from:
wg1.b.yahoo.com
    Origin: yf1.yahoo.com

```sh
nslookup new-fp-shed.wg1.b.yahoo.com yf1.yahoo.com
fg
^C
```

answer doesn't note about being a non-authoritative response


`sudo tcpdump -t -n -r /tmp/dns.pcap`

## Now, let's resolve a name top-down

First ask the root nameserver...
query(_.com, .)
- we used to ask root with the whole query, telling everyone what we're searching for. (not private)
- nowadays we have DNS query name minimization where they strip off all labels and send a more private query.

Root nameserver replies with NS records responsible for the TLD you queried about. It also is helpful and provides
the IP addresses in A and AAAA records in the same response packet in the "additional" section.

query(_.yahoo.com, a.gtld-servers.net)
The .com nameserver can tell use where to find yahoo.com nameservers. It also tells us the IP addresses.

query(www.yahoo.com, ns1.yahoo.com)
This nameserver tells us www.yahoo.com is a CNAME and that we need to talk to yf1.yahoo.com

query(new-fp-shed.wg1.b.yahoo.com, yf1.yahoo.com)
we finally get the authoritative answer.

Caching resolvers will save these results so you don't have to keep making these queries.

It is easy to change our EC2 instance into a caching resolver on NetBSD:
```sh
echo "named=YES" | sudo tee -a /etc/rc.conf >/dev/null
vi /etc/resolv.conf
> nameserver ::1 # use local host as resolver
# start tcpdump
sudo /etc/rc.d/named start # this is the BIND nameserver
sudo rndc flush # flush caches
nslookup www.yahoo.com
```

There's TCP and UDP traffic as well as other RR types we didn't discuss. This will be important
for the next homework.

Why is there TCP traffic?
What are the root nameservers?

# What is Root?

Getting the nameservers responsible for the root zone:
`dig -t ns .`

There are 13 root nameservers in total (a-m). They neatly fit into a single 512 byte udp packet.

Nowadays, with the additional section, the size goes over 512 bytes, so the resolver will try to query over TCP.
The additional section contains the IP addresses of these nameservers, which are all dualstack.

This information is bootstrapped for every nameserver installation. Otherwise, there is a chicken-or-egg problem.
Under /etc/namedb/root.cache, we find that information.
This file is part of the `named` software distribution (BIND). Also available on the Internet via FTP from the InterNIC servers.

# How many root nameservers?

There are more than 13 nameservers!!! So we can't just DDoS 13 of them and take down the internet.

What we saw are the 13 root nameserver *authorities*. (a root, b root, ... m root)
Each of these root authorities are comprised of dozens or hundreds of individual servers.

https://root-servers.org

The 13 roots are operated by 12 independent and international organizations.
- no single country has control over the domain name space.

Here in NY, we have 9 root server instances: A,C,D,E,F,J

Not all roots have servers in the same locations.

In Nuuk, Greenland, there is a single K root operated by RIPE.

Many nameservers are colocated with the central network hubs.

E root is operated by NASA and is colocated all across the globe in over 160 locations

B root is operated by the University of Southern California, but served from peering points across the world.

Root server operation is not a business. It is provided as a public service.

C root is run by Cogent, a commercial business.

F root is run by ISC.

The root servers are all using "anycast" to allow multiple systems to offer the same IP address and to let the routing
algorithms to determine the closest one, generally as determined by number of hops.

How do you know which actual server you end up talking to?
- it doesn't matter, but you can `dig +norec @f.root-servers.net hostname.bind chaos txt`
- this is asking the F root for the hostname.bind text record using the chaosnet protocol.
- the response's first label has a 3 letter airport code. our F root is close to Dulles Intl. Airport in DC.

^ DNS is a distributed database

# Different RR records

Using Text records, we can put any information into the DNS that we want.

NS records use the internet (IN) class of records. Each resource record also has a ttl.
It describes how long a caching resolver may cache the results for.

The NS records for the root nameservers are long-lived (~5 days)

Start of Authority (SOA) record
- defines information about the zone.
- `host -t soa stevens.edu`
- name of primary master, email of primary point of contact, serial number, refresh timeout for secondary nameservers,
  retry, expire, and ttl number.
- The authoritative nameservers for a domain don't have to be under that domain.
  `host -t ns stevens.edu` -> shows azure-dns domains from multiple TLDs (redundancy)
- `host -t soa stevens-tech.edu` -> different from stevens.edu

Certificate Authority Authorization (CAA) record
- determines which CAs are allowed to issue certificates for the given domain
- `host -t caa yahoo.com` -> yahoo.com only allows DigiCert and GlobalSign.

Text (TXT) record
- `host -t txt istheinternetonfire.com` -> some random text

SSHFP record
- you can put your host's ssh key fingerprints so that clients can verify who they're connecting to.
- `host -t SSHFP cs615asa.netmeister.org`

The most common lookups are mappings from domain to ip and vice versa.

The reverse lookup of domains from IP addresses is interesting. It uses different domains.
- it uses the PTR record
- `host -t ptr 155.246.56.11`
- output of the command reverses the bytes of the address and appends in-addr.arpa, 11.56.246.155.in-addr.arpa
- how does in-addr.arpa know?
- the chain of queries is similar to www.yahoo.com
  it allows us to effectively do a whois lookup by netblock.
  - who has 155.in-addr.arpa -> nameserver run by ARIN (155/8)
  - who has 246.155.in-addr.arpa -> nameserver run by Stevens (nrac.stevens.edu)
  - etc.

reverse IP lookups work almost the same as normal DNS.

# DNSSEC

dig www.iana.org returns `ad` flag meaning authenticated data.

```sh
sudo tcpdump -w /tmp/dns.pcap port 53 >/dev/null 2>&1 &
sudo rndc flush # flush caches
nslookup www.yahoo.com
dig www.iana.org
fg
^C
```

Resource Record Signature (RRSIG) record
- cryptographic signature asserting the integrity of the entire NS record set.

Delegation Signer (DS) record
- used to identify the signing key of the zone.

Final request also includes an RRSIG. How to verify?
- get public key with DNS! It is a distributed lookup table.
- Information didn't fit into one UDP packet, so truncation occured.
- Transmission using TCP instead.

System of signing records by different zone keys is known as the
Domain Name System Security Extensions

# DNS Takeaways

DNSSEC is not widely deployed (unfortunately)

If you mess up DNSSEC, your entire domain falls off the internet.

Look at:
DNS over TLS (DoT)
DNS over HTTPS (DoH)

DNS traffic is ubiquitous, may escape ACLs and restrictions.

Misconfiguration can lead to endless hours wasted.
Troubleshooting can be tricky due to the nature of
DNS, allowing for delegation and different TTLs on records.

TTLs and caches can prolong outages as you wait for propagation of changes

"If everything is all messed up and doesn't make sense, it's probably the DNS"...
- or someone thought it was clever and tried modifying /etc/hosts...

if you pwn the DNS, you pwn the entire target.

any time you outsource, you lose control

any time you own solving a problem, you assert that you know how to solve this better than others.

---

# Checkpoint

> What protocol and well-known port does the DNS typically use?

UDP on port 53

> Specify four DNS record types (RRs) besides A and AAAA.

A - Address (IPv4)
AAAA - Address (IPv6)
NS - Name Server
MX - Mail Exchange
CNAME - Canonical Name
SOA - Start of Authority
CAA - Certificate Authority Authorization
TXT - Text
SSHFP - SSH Fingerprint
PTR - Pointer
RRSIG - Resource Record Signature
DS - Delegation Signer

> How many top-level domains are there today?

`curl -s ftp://rs.internic.net/domain/root.zone | grep "IN\tNS\t" | awk '{print $1}' | sort -u | wc -l`
1444 as of March 2025

> How many 'root servers' are there? Provide at least two IP addresses.

`curl -s https://root-servers.org | grep "As of" | cut -c 32-`
According to https://root-servers.org, there are 1907 instances.

Aberdeen Proving Ground, US (H): 2001:500:1::53
Shenyang, CN (I): 2001:7fe::53

> When/why do we use TCP instead of UDP for DNS traffic?

When the data that is stored exceeds the size of a UDP packet, DNS traffic will
be sent over TCP. This usually happens because of the Additional Section.

> On a caching resolver, what is the first query when it tries to resolve www.cs.stevens.edu?

"A?" for www.cs.stevens.edu

> On a caching resolver, what is the first query when it tries to resolve the PTR record for 155.246.56.11?

"PTR?" 11.56.246.155.in-addr.arpa.

> What threats do DNS-over-TLS and DNS-over-HTTPs address that DNSSEC does not?

Both solutions deal with the threat of trackers by implementing end-to-end
encryption around DNS so that no one can read the traffic.

# In Class

## DNS

```sh
dig www.stevens.edu a
dig +dnssec www.stevens.edu a
dig +dnssec stevens.edu dnskey
dig +dnssec stevens.edu ds
dig +dnssec edu. dnskey
dig +dnssec edu. ds
dig +dnssec . dnskey
delv +vtrace www.stevens.edu a

https://dnsviz.net/
```

quad9 `9.9.9.9` DNS

dns.google

ECS is the Client Subnet Extension

Many ccTLDs do not have DNSSEC enabled

ISPs can monetize DNS traffic by showing ads for missing websites.

Browser implements the DNS over HTTP. It is not standardized, so different addresses might be returned.

CHECK SLIDES FOR THE CODE I MISSED for DoT and DoH
```sh

```

Resource Records

IN - internet protocol
- ns

CH - chaosnet protocol
- txt
- `dig ch txt version.bind`
- `dig ch txt authors.bind`
- `dig txt ch authors.bind` same as above
- `dig ch ns` doesn't work
- `dig ns ch.` looks for ch. fqdn
chaosnet was an earlier protocol at MIT.

Hesiod (also at MIT - Athena project)
- distributed database - authentication
- `dig hs txt jschauma.passwd.hs.dotwtf.wtf`
- `dig hs txt 100.gid.hs.dotwtf.wtf`
look up information about a system

TTL timeouts and propagation

Some sites have TTL of an hour.
CDNs usually change the timeout to a much smaller number.

Why is www prepended to most links?
- www is just random choice for "World Wide Web"
- You CANNOT have a CNAME at the apex of a zone: google.com cannot be a CNAME
  - you have a NS record

Why doesn't Stevens (stevens.edu) have any CAA records?
- we're outsourcing.
- cloudflare requests certificates as needed.
- probably no one did it... (we don't know)
- CAA records have been around for 10-ish years
- Jan being sysadmin at Stevens predates that and it was for the CS department specifically.

how are DNS records stored in the servers such that they can be looked up efficiently?
- distributed database problem
- also load-balanced, or maybe use anycast

## HTTP

Tim Berners-Lee
CERN
NeXT-Box

1991: 0.9
1996: 1.0
1997: 1.1
1999: 1.1 updates
2012: QUIC
2015: 2
2021: 3

---

HTTP:
- CGI - resource is executed, needs to generate appropriate response headers
- serverside scripting
- clientsite scripting


Scaling HTTP Traffic

mitigate HTTP overload
- DNS round-robin to many web servers
- load balancing
- web cache / accelerators (CDNs)

CDNs:
- cache content in strategic locations
- determine location to serve from  via geomapping of IP addresses
- either out-sourced (Akamai, Cloudflare, Fastly, ...), in-house, or both
- request routing via Global Server Load balancing, DNS-based request routing, anycasting
- ...
- your CDN sees all your traffic
- your CDN controls your TLS cert keys
- your CDN is a multi-tenant environment
- your CDN may impose restrictions on your clients, content, ...
- your CDN may be subject to different jurisdictions
- you lose control of the edge





