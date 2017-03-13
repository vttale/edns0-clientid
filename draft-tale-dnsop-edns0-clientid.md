---
title: Client ID in Forwarded DNS Queries
docname: draft-tale-dnsop-edns0-clientid-01
date: 2017

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: R. Licht
    name: Robert Licht
    org: Charter Communications
    street: 13820 Sunrise Valley Dr
    city: Herndon
    code: VA 20171
    country: USA
    email: robert.licht@charter.com

  -
    ins: D.C. Lawrence
    name: David C Lawrence
    role: editor
    org: Akamai Technologies
    street: 150 Broadway
    city: Cambridge
    code: MA 02142-1054
    country: USA
    email: tale@akamai.com

normative:
  Address_Family_Numbers:
    author:
        name: IANA
    title: Address Family Numbers
    target: http://www.iana.org/assignments/address-family-numbers/

informative:
  DPRIVE_Working_Group:
    author:
      -
        name: Warren Kumari
        ins: W. Kumari
      -
        name: Tim Wicinski
        ins: T. Wicinski
    title: DPRIVE Working Group
    target: https://datatracker.ietf.org/wg/dprive/charter/

  dnsmasq:
    author:
      name: Simon Kelley
      ins: S. Kelley
    title: dnsmasq
    target: http://www.thekelleys.org.uk/dnsmasq/doc.html

  Vantio_CacheServe:
    author:
      organization: Nominum, Inc.
    title: Vantio CacheServe
    target: http://www.nominum.com/product/caching-dns/

  Umbrella:
    author:
      org: Cisco Systems, Inc.
    title: Umbrella
    target: https://docs.umbrella.com/developer/networkdevices-api/identifying-dns-traffic2

--- abstract

This draft defines a DNS EDNS option to carry a client-specific
identifier in DNS queries, with guidance for privacy protection of
such information.

--- note_Ed_note

Text inside square brackets (\[\]) is additional background
information, answers to frequently asked questions, general musings,
etc.  They will be removed before publication.  This document is being
collaborated on in GitHub at
\<https://github.com/vttale/edns0-clientid\>.  The most recent
version of the document, open issues, etc should all be available
here.  The authors gratefully accept pull requests.

--- middle

# Introduction

Some DNS operators generate, or wish to generate, customized DNS
responses based on the originator of a DNS query.  For example,
{{!RFC7871}}, "Client Subnet in DNS Queries", defines a method to
convey partial IP network address information about the device that
originated a DNS request, so that a response could be targeted to be
topographically near the source.

Some specialized services, however, need more precise client identity
information to function adequately.  For example, a parental control
service that restricts access to particular domains from particular
devices needs to have a device-specific identifier.

This document defines an EDNS {{!RFC6891}} option to convey client
identification information that is relevant to the DNS query.  It is
added by software on the client's local area network, for transmission
to the upstream DNS provider.

A similar EDNS option is already being used on the public Internet in
two different implementations.  One is between the {{dnsmasq}}
resolver on the client side and Nominum's {{Vantio_CacheServe}}
upstream.  It uses EDNS option code 65073 from the "Reserved for
Local/Experimental Use" range to pass the client's Media Access
Control (MAC) address.  The other implementation is for Cisco's
{{Umbrella}}, aka OpenDNS, which encodes the client's MAC address and
complete IP address.  It uses option codes 26946 and 20292,
respectively, from the middle of the "Unassigned" range.

This document codifies a more flexible format that can accommodate the
needs of both implementations, as well as other more opaque
identifiers.  It is intended to supersede those non-standard options.

This option is intended only for constrained environments where its
use can be carefully controlled.  It is completely optional and should
be ignored by most DNS software.

# Privacy Considerations

The IETF is actively working on enhancing DNS privacy
{{DPRIVE_Working_Group}}, and the re-injection of personally
identifiable information has been identified as a problematic design
pattern {{?I-D.hardie-privsec-metadata-insertion}}.

Because this option transmits information that is meant to identify
specific clients, to be considered compliant with this draft
implementations MUST NOT add the option without explicit opt-in by an
administrator on the local area network.  For example, agreeing to the
terms of service for a device-specific DNS filtering product would
allow the option to be enabled, and only for communication to the
product's DNS server(s).

Implementers need to be aware of the various laws and regulations
governing handling personal data, but they are out of scope of this
document.

No explicit provision is made in the protocol to opt-out.  For more
discussion on this, see {{security-considerations}}, "Security
Considerations".

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}

For a comprehensive treatment of DNS terms, please see {{!RFC7719}}.
This document uses the following additional terms:

ECID
: EDNS Client Identification.

Client
: The user or device that originates a DNS lookup.

Nameserver
: A DNS server capable of resolving a DNS query and formulating a response.

Forwarding Resolver
: A nameserver that does not do iterative resolution itself, but
instead passes that responsibility to another resolver, called a
"Forwarder" in {{!RFC2308}} section 1.

Tailored Response
: A response from a nameserver that is customized based on a
policy defined for the client requesting the query.

# Option Format

This protocol uses an EDNS {{!RFC6891}} option to include client
identification information in DNS messages.  The option is structured
as follows:


                  +0 (MSB)                        +1 (LSB)
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |                         OPTION-CODE                       |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                        OPTION-LENGTH                      |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    4: |                       IDENTIFIER-TYPE                     |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    6: |                                                           /
       /                      CLIENT-IDENTIFIER                    /
       /                                                           /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

OPTION-CODE:
: 2 octets per {{!RFC6891}}.  For ECID the code is TBD by IANA.

OPTION-LENGTH:
: 2 octets per {{!RFC6891}}.  Contains the length of the payload
following OPTION-LENGTH, in octets.

IDENTIFIER-TYPE:
: 2 octets per {{Address_Family_Numbers}}, describing the format of
CLIENT-IDENTIFIER as elaborated below. \[ Is it better to call this
ADDRESS-FAMILY? \]

CLIENT-IDENTIFIER:
: A variable number of octets, depending on IDENTIFIER-TYPE.

All fields are in network byte order ("big-endian", per {{!RFC1700}},
Data Notation).

This draft only specifies behaviour for the following IDENTIFIER-TYPE
values and the corresponding CLIENT-IDENTIFIER lengths:

* 16389 (0x4005, 48-bit MAC): 6 octets, fixed.
* 1 (0x0001, IP version 4): 4 octets, fixed.
* 2 (0x0002, IP version 6): 16 octets, fixed.
* 16 (0x0010, Domain Name System): Variable-length domain name in
uncompressed wire format followed by a variable-length custom token.

For DNS servers that implement ECID, it is RECOMMENDED that they
recognize at least the 48-bit MAC CLIENT-IDENTIFIER.

The use of Domain Name System as an address family is to facilitate
custom tokens that are not well-conceptualized as addresses, as
described in {{using-the-dns-address-family}}.

Other types of identifying addresses, such as a 64-bit MAC
{{?RFC7042}} or a DHCP Unique Identifier {{?RFC3315}} and {{?RFC6355}}
could be accommodated as devices and needs change, without needing to
define new EDNS option codes to cover them. \[ Why not just
bless those obvious candidates now? \]

Multiple ECID options MAY appear in the OPT record.  However, the same
IDENTIFIER-TYPE SHOULD not appear more than once, and each ECID option
MUST only carry one IDENTIFIER-TYPE and CLIENT-IDENTIFIER pair.

# Protocol Description

## DNS Query

Any client that originates a DNS query message MAY include the
ECID option in the DNS Query message.  It is normally expected that
the client itself would not do this, but rather that it will be added
by the local forwarding resolver.

When a DNS forwarding resolver, provided as part of a router for
example, receives a DNS query message from the originating client it
adds any IDENTIFIER-TYPE / CLIENT-IDENTIFIER pairs that it supports but
which are not present in the existing client request.  It then sends
the request to the upstream full-service resolver.

Because the option contains personally identifiable information, it
should be protected by either only being used within Autonomous
Systems {{!RFC1930}} controlled by the same provider, by going over an
opaque channel such as DNS over TLS {{!RFC7858}}, or by being securely
encoded and varying per request.  It MUST NOT be sent in clear-text
across the Internet.

## DNS Response

The logic used by a full-service resolver to customize a response
based on ECID is out of scope for this document.  The resolver MUST
NOT include the ECID option in any queries that it makes to external
authoritative DNS servers.

For possible caching purposes, the forwarding resolver needs to know
whether filtering affected the response.  If the name resolution
involved any names for which customization was possible, even if such
filtering resulted in delivering the original data, the response
SHOULD include an ECID option which contains the FAMILY-ADDRESS and
CLIENT-IDENTIFIER pairs that were considered for filtering.

For example, if a filter is set such that only names in the
example.com domain are possibly restricted to some devices, then a
request for foo.example.com would have the ECID in the response even
when the request came from a device which was not restricted.
Requests for any other names would not include ECID in the response.

So that the caching forwarding resolver does not need to have any
knowledge about what filters are in place, it is the full-service
resolver's responsibility to adjust any TTLs in the response as might
be dictated by the filter policy it has configured.  That is, if some
name is filtered only between the hours of 09:00 and 17:00 and a
request is received for that name at 16:59:59, the TTL on a positive
response or the SOA ncache field on a negative response should be set
to just one second and the ECID option included as described above.

If the request contains a malformed ECID option, such as
CLIENT-IDENTIFIER not correctly matching the length of described by
OPTION-LENGTH and IDENTIFIER-TYPE, the resolver SHOULD reply with DNS
rcode FORMERR.

If the resolver by policy does not respond to requests that are
lacking ECID of the appropriate IDENTIFIER-TYPE, it SHOULD reply with
DNS rcode REFUSED.

# Using the DNS Address Family

When IDENTIFIER-TYPE 16 is used, the uncompressed wire format of the
domain name is followed by a token that is otherwise opaque to this
specification.  The length of that token is defined by OPTION-LENGTH
less the two octets used for IDENTIFIER-TYPE and the length of the
domain name.

The name used SHOULD be in a namespace that is controlled by the
service provider that is using the option, but need not be resolvable
in the DNS.  We RECOMMEND that providers use short domain names to
minimize DNS packet length.

The domain name provides protection against conflicts with other users
of the option without the burden of creating yet another IANA Registry
to manage yet another two-octet code. Co-operating forwarder/resolver
pairs are the only users of the data who need to be concerned with its
format.

# Implementation Status

\[RFC Editor: per RFC 6982 this section should be removed prior to
publication.\]

The protocol proposed here is not currently used anywhere exactly as
described, though the Nominum and Umbrella implementations are
substantially similar.

The authors know of at least two providers who wish to have it
properly standardized and would implement the standard in preference
to either of the existing non-standard methods.

# NAT Considerations

Devices that perform Network Address Translation (NAT) SHOULD NOT give
special consideration for ECID. NAT translates between a layer 3
private IP address assigned to a client device on the Local Area
Network and a layer 3 public IP address for use within the Wide Area
Network.  If ECID is being used to pass an IPv4 or IPv6 address, it
SHOULD use the internal address without NAT translation, because
transforming it to the public address of the NAT device would coalesce
all internal devices to just one address.

Other ECID options identify a client device by a different means,
e.g. its layer 2 address. This sort of device's identifier is not
impacted by NAT.  Therefore, DNS queries may be passed without
modification of any ECID information.

# Security Considerations

The identifier of the client that initiated the request will be
visible to all servers that are passed the ECID option, and the
various devices on the path between the local network and the
full-service resolver being used by the forwarding resolver.

DNS filtering products are easy circumvented and should not be
considered real security measures.  With commonly available tools it
is trivial to discover the non-filtered DNS responses and use them in
place of the filtered responses.

Along those lines, opting out of this specific protocol is as simple
as using a different resolver, such as a full-service resolver on the
device itself or one of the well-known public resolvers.  Of course,
other devices on the local network will still be able to see
unencrypted DNS requests from the device, and the only way to really
protect against such monitoring is to use an opaque tunnel to a
trusted resolver.

# IANA Considerations

IANA is requested to assign a new value in the DNS EDNS Option Codes
registry for the Device ID option.

# Acknowledgements

The authors wish to thank the Barry Greene, Martin Deen, Benjamin
Petrin, and Robert Fleischman for their feedback and review during the
initial development of this document.

--- back
