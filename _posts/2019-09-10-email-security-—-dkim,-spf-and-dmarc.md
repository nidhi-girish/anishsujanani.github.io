
With email being one of the largest forms of communication over networked computers, there is an obvious importance in understanding the underlying protocols, pitfalls and security measures. This article documents my understanding of the current best-practice email security frameworks.

## DKIM — Domain Keys Identified Mail
An application of the PKI crypto-system, DKIM is a control that detects mail tampering during transit through verification of hashes. The sender mail transfer agent (MTA) hashes certain headers of the mail and signs this hash using its private key. The DNS hosting for this sender MTA holds the MTA’s public key as a TXT record. On receiving the mail, the receiving MTA hashes the same headers of the incoming mail and also queries DNS nameservers for the sender MTA’s public DKIM key. The receiving MTA decrypts the signed hash from the sender, and compares it to the hash it just computed. A difference in hashes implies that somewhere along the way, the mail was tampered with and should be discarded/alerted on.

### How do sender and receiver MTA(s) decide on which email headers to hash? How is the type of DNS record looked up? Which record name is to be queried for? How is the hash actually transferred?

The sending MTA will attach an additional DKIM header to the mail which contains all the above information and more to implement this control:

`DKIM-Signature: v=1; a=rsa-sha256; d=senderdomain.com; s=pubkeyselector; c=relaxed/relaxed; q=dns/txt; t=1286402482; x=1441825284; h=from:to:subject:date:keywords:keywords;
bh=ff35OeIzK1jcDUHPzRjNMTAy2NzDEM0BLMUyNY3ODlq=;
b=hyjwCnONifAKrDdKIF9G1q7LoDWlEniSbdLZzc+yuU2zGtr0cu0ldcF
VoG4WTHYG`

The above fields are explained as follows:
- `v=1`: Version of the DKIM signature spec
- `a=rsa-sha256`: Signature Generation Algorithm
- `d=senderdomain.com`: Domain name of sender, to be used with ‘s’
- `s=pubkeyselector`: Selector to query for from above
- `c=relaxed/relaxed`: Minor modifications to the formatting of the hashes that differs between MTA(s), eg. whitespace and line wraps — formatting differences that should not invalidate the signature
- `q=dns/txt`: Type of record to query for
- `t=1286402482; x=1441825284`: Timestamp and expiration for the signature
- `h=from:to:subject:date:keywords:keywords`: Headers to concatenate and hash
- `bh`: Hash of the body of the email, b64 encoded
- `b`: Hash of the data represented by h — the headers of the email, b64 encoded

The spec [RFC6376](https://tools.ietf.org/html/rfc6376) dictates additional fields and their purpose.

----------------

## SPF — Sender Policy Framework
Let us assume a scenario where a bad actor starts spamming mail out to a mailing list using a custom script. Email may be bounced off an open relay, while spoofing the ‘from’ address to your domain. The underlying SMTP protocol does not have address-domain validation mechanisms and the mail will go through as long as the open relay allows it. Your domain will now start growing its spam score across multiple email service providers, affecting your domain’s communication severely and possibly disrupting legitimate interaction causing severe business impact.

SPF, like DKIM relies on DNS as a backbone for its implementation.

Let us assume the following configuration:
- Email sent from IP Address aa.bb.cc.dd with an envelope-from: `mailadmin@alloweddomain.com`.
- The recipient of the mail: `bob@actualuser.com`
- Your domain’s DNS provides a TXT record specially formatted to match SPF spec:
`v=spf1 +a +mx -all`

Once the mail server at `actualuser.com` receives this email, it will look up the SPF record for the sender domain: `alloweddomain.com`. This query will return the SPF record string listed above.

- `v=spf1`: Specifies which version of the parser to use
- `+a`: Look up the A record for the sender domain alloweddomain.com. If this query returns the IP address that sent the mail: aa.bb.cc.dd, continue.
- `+mx`: Look up the MX record for the sender domain alloweddomain.com. If this query returns the IP address that sent the mail: aa.bb.cc.dd, continue
- `-all`: Drop all email that do not match the prior A record and MX record criteria
- `Support for IPv4/IPv6 addresses`: +ip4:aa.bb.cc.dd

There is another include clause that can be used with SPF records. It seems a bit more complex but from my understanding, it can be used to chain SPF record validation across multiple domains. eg. `+include:otheralloweddomain.com` would carry out the entire SPF record validation for `otheralloweddomain.com` and return a PASS/FAIL value. This can be used for allowing mail transfer through third party services.

In the example above, we use only the “+” qualifier to allow. Other qualifiers include:
- `-`: Disallow
- `~`: Allow the mail to go through but mark an SPF failure
- `?`: Allow the mail to go through, no pass/fail value for the SPF procedure

A downside of SPF validation is the failure of automated mail forwarding. Depending on your record, it is possible that forwarded mail(s) are dropped. Mail servers can be configured to maintain the sender_email address in the envelope section as a workaround to this.

------------------

## DMARC — Domain Message Authentication, Reporting and Conformance
In a nutshell, DMARC = DKIM + SPF + Additional Reporting and Policy creation/enforcement features.

DMARC — also implemented as a TXT DNS record allows mail servers to be configured with policies on how to handle mail authentication through one or both of the above mechanisms — SPF and DKIM, along with specification on forwarding, dropping and quarantining mail.

DMARC also gives you further flexibility on your SPF and DKIM policy. Below is a sample DMARC record.

`v=DMARC1; p=quarantine; sp=none; adkim=s; aspf=r; pct=100; rua=mailto:admin@yourdomain; ruf=mailto:admin@yourdomain; fo=1;`

- `v=DMARC1`: DMARC version
- `p=quarantine`: What to do when DMARC validation fails? none/quarantine/reject
- `sp=none`: What to do when DMARC validation fails on subdomains? none/quarantine/reject
- `rua=mailto:admin@yourdomain`: mailbox to send aggregate reports on protocol
- `ruf=mailto:admin@yourdomain`: mailbox to send failure reports on failed messages
- `fo=1`: Type of failure reports to generate. s=spf failure reports, d=dkim failure reports, 1=if any mechanisms fail, 0=if all mechanisms fail
- `adkim`: Policy for DKIM validation. s : strict mode — the sender domain in the mail headers should exactly match the DKIM header. r : relaxed mode — sender domain in the mail - headers can be a subdomain of the domain in the DKIM header. Let’s say the DKIM header contains somedomain.com. The mail_from header contains `user@users.somedomain.com`. The mail will fail DKIM validation under strict mode, but pass under relaxed mode.
- `aspf`: Policy for SPF validation. Similar to the above adkim, it allows both strict and relaxed modes to allow for either exact matches or subdomain matches between SPF records and mail envelope header senders.

--------------------------

As seen above, DMARC is a framework that ties together DKIM and SPF validation, to provide a holistic email security control that is able to:
- Detect tampering of email - DKIM
- Prevent email sender spoofing, contributing to maintaining a healthy spam score and domain reputation - SPF
- Extending the above for main domains and subdomains, granularity of control, reporting and analytics - DMARC

It is particularly interesting to me how all three of the above controls rely on DNS as an implementation backbone.
