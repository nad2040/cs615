# Email

Not dead yet.

5.5 billion email accounts

300 billion emails per day

Mail User Agent - program that allows you to read your emails (mutt, Outlook)

Mail Transfer Agent - mail server (postfix, sendmail, qmail) - outgoing and receiving ends

Mail Delivery Agent - procmail
Access Agent - (POP, IMAP)

## Sending end

SMTP (RFC821)
- transport in clear text (port 25)
- use of DNS MX records for service discovery
- text based protocol with simple dialog structure and return codes

```
HELO / EHLO
MAIL FROM
RCPT TO
DATA
```

```sh
sudo tcpdump -w /tmp/smtp.pcap port 53 or port 25 >/dev/null 2>&1 &
mail -s "SMTP Test" jschauma@netmeister.org
# EMAIL text here
.

# mail server logs into /var/log/maillog
```

server seem to reject messages if there's no dns record

procmail does some spam assessment

manually send email using telnet
```
host -t mx netmeister.org
host panix.netmeister.org
telnet <ip> 25
...
EHLO cs615asa.netmeister.org
...
MAIL FROM: <jschauma@cs615asa.netmeister.org>
RCPT TO: <jschauma@netmeister.org>
DATA
From: Jan Schaumann <jschauma@cs615asa.netmeister.org> # These are different headers.
To: jschauma@netmeister.org
Subject: SMTP Test

Hello,

SMTP is a _simple_ mail transfer protocol, indeed.

-Jan
.
quit
```

## Receiving End

Using STARTTLS

```
telnet <ip> 25
...
ehlo cs615asa.netmeister.org
...
STARTTLS # SMTP over TLS
how do you even write tls?????
```

use
```
openssl s_client -connect panix.netmeister.org:25 -4 -starttls smtp
ehlo <ec2-domain>
...
mail from: <jschauma@ec2-domain>
...
rcpt to: <jschauma@netmeister.org>
...
data
...
.
```
now, traffic is secure

Check whether a domain wants to use TLS for all its MTAs:
`host -t txt _mta-sts.netmeister.org`
mta-sts = mta strict transport security

if there is a policy, retrieve the file https://mta-sts.netmeister.org/.well-known/mta-sts.txt

How do we know if we can trust the certificate?
- if you remove the certificate, you can try a different choice:

`dig _25._tcp.panix.netmeister.org tlsa`
- this record is part of the DNS-based Authentication of Named Entities (DANE) standard
- add to the openssl command
```
-dane_tlsa_domain panix.netmeister.org
-dane_tlsa_rrdata "output from dig after the TLSA in the answer section no space between the hex parts"
```

- STARTLS may be used to add encryption in transit
- STARTTLS may be used to add authenticity guarantees to the client
- MitM can strip STARTTLS
  - SMTP MTA Strict Transport Security (MTA-STS) (RFC8461)
- MitM can present fraudulent certificate
  - solution, ignore pki certs
  - DNS-Based Authentication of Named Entities (DANE) (RFC7672)
- Should failure to verify certificate lead to mail to being delivered?

# Spam Detection

Spam Assassin detects spam with a couple rules
- missing headers, etc.

## SMTP Headers
- Message-Id
- Date

Message-Id didn't display. Ask mail client to show all headers.
we see
- Received:
- Received-SPF:

Mandatory Headers:
- From
- Date:

Optional Headers:
- From:
- To:
- Subject:
- ...

Content of Message:
- is independent of SMTP
- MIME enables non-ascii, multipart, encodings, ...

## Mail Relay

Services choose to not relay email for domains they don't control

In the old days, MTAs would accept and relay email from anybody to anybody

So, we gotta lookup the MX dns for the domain we want.
```sh
host -t mx stevens.edu
```

## Spoofing "From"
SMTP provides no authenticity guarantees
"From " can be set to anything
"From: " can be different from "From "

## Sender Policy Framework (SPF)

Yahoo doesn't accept the spoofed mail.

SPF allows domain owner to specify which systems are allowed to send email on your behalf.

`host -t txt _spf-a.microsoft.com | grep spf`
there's ~all

`host -t txt netmeister.org | grep spf`
```
v=spf1 a mx -all
```

-all and ~all are different:
- -all means hard failure
- ~all means soft failure
- they don't mean mail has to be rejected. it is an indicator to the mail server that it should consider rejecting.

## Forwarded Email and "Received:" Header

Received: headers trace the path our email took to reach our mailbox.

How do I know the received message is true?
- Authentication-Results: header is useful
- your message can be signed

## DomainKeys Identified Mail (DKIM)

can help detect email spoofing by providing a digital signature across parts of the message.

combines efforts by Yahoo (enhanced DomainKeys) and Cisco (Identified Internet Mail)

adds DKIM-Signature headers

more DNS TXT records (`<s>._domainkey.<domain>`)

multiple "Authentication-Results:" and "DKIM-Signature:" headers exist for the hops

A DKIM-Signature contains info about:
- which domain it is responsible for (d=...)
- a "selector" to identify the correct public key (s=...)
- hash of the body of the email (bh=...)
- the signed header fields (h=...)
- the signature of the data (b=...)

For validation, retrieve the correct pubkey via the DNS by combining the selector, "_domainkey" and the domain.

What is DMARC?
## Domain-based Message Authentication, Reporting and Conformance

DMARC provides a policy of which validation mechanisms should be employed for a given domain.
- uses SPF and DKIM
- extends across "From " and "From:" alignment
- provides report mechanism
- more DNS TXT records (`_dmarc.<domain>`)

DMARC policy tells you what to do:
- reject
- accept but quarantine
- ...

- SPF checks that SMTP MAIL FROM is authorized; DMARC ensures alignment with "From:"
- DKIM allows parts of the message to be signed; DMARC allows the domain owner to specify what to do if e.g., the signature is wrong
- DMARC allows for aggregate reports of failed attempts

## Summary

SMTP headers:
- some are mandatory, some optional
- lack of some may be used as a signal of spamminess
- each hop may add additional headers

SPAM protections:
- recipient restrictions (no open relays)
- sender IP reputation (e.g., via DNS lookups in community databases)
- Sender Policy Framework (SPF) specifies who is authorized to send mail on a domain's behalf
- DomainKeys Identified Mail (DKIM) signs parts of the mail
- DMARC lets responsible domain specify what recipients should do upon mismatches

Email Service Implications
- spam protections
- phishing protections
- high volume traffic demands fine-tuned systems
- high volume traffic implications on logging
- mail delivery cannons for notifications vs. spam lists
- outsourcing versus in-house
- privacy considerations

# Checkpoint

> How does HTTPS differ from HTTP?

HTTPS is just a secured HTTP protocol with TLS.

> How does a user procure an x509 certificate?

You obtain an x509 cert by sending a Certificate Signing Request (CSR) to a
Certificate Authority (CA). The CA verifies that information and then returns
the certificate signed with its private key.

> How does a client validate the certificate presented by a server?

The client can verify through the chain of certificates back to the root CAs
that a given certificate is valid.

> What are some pitfalls of HTTPS?

Incorrect HTTPS deployment trains users to ignore errors

Controlling a CA gives you near universal powers

> What protocol and well-known port is usually used for SMTP?

SMTP is built on the TCP protocol over port 25

> Identify at least two mail headers commonly hidden from the user. What are their purposes?

Received-SPF: - shows the result of the Sender Policy Framework check
Message-Id: - uniquely identifies an email message

> What mechanisms exist to ascertain an email's authenticity?

Sender Policy Framework (SPF)
DNS-Based Authentication of Named Entities (DANE)
Domain-based Message Authentication, Reporting and Conformance (DMARC)

> What mechanisms exist to ensure an email's confidentiality?

STARTTLS - encryption with TLS

# In Class

cloudfront has redundancy w/ multiple nameservers under different TLDs.

The requests for the root servers are updated just in case.

POP doesn't sync sent mail.
IMAP does a pull down from the server.


TLS is a set of cryptographic protocols

ClientHello - present list of supported cipher suites
ServerHello - choose cipher suite
ServerCertificate -
(Server Key Exxchange Message, Client Certificate Request, Client Certificate)
Client Key Exchange Message
(Certificate Verify)
(Client Change Cipher Spec, Server Change Cipher Spec)

https://netmeister.org/blog/tls-hybrid-kex.html

ca/browser forum

Abuse email: abuse@domain


