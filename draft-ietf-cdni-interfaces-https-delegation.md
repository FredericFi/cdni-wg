---
v: 3

title: CDNI extensions for HTTPS delegation
abbrev: CDNI extensions for HTTPS delegation
docname: draft-ietf-cdni-interfaces-https-delegation-latest

category: std
consensus: true
submissiontype: IETF

ipr: trust200902
area: ART
workgroup: CDNI Working Group
keyword: [HTTPS, delegation, ACME, STAR]

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 - name: Frédéric Fieau
   org: Orange
   street: 40-48, avenue de la Republique
   city: Chatillon
   code: 92320
   country: France
   email: frederic.fieau@orange.com
   role: editor

 - name: Emile Stephan
   org: Orange
   street: 2, avenue Pierre Marzin
   city: Lannion
   code: 22300
   country: France
   email: emile.stephan@orange.com

 - name: Sanjay Mishra
   org: Verizon
   street: 13100 Columbia Pike
   city: Silver Spring
   code: MD 20904
   country: United States of America
   email: sanjay.mishra@verizon.com

venue:
  group: Content Delivery Networks Interconnection
  mail: cdni@ietf.org
  github: TODO

normative:
  RFC8006:
  RFC8008:
  RFC8739:
  RFC9115:

informative:
  RFC7336:
  RFC7337:

entity:
  SELF: "RFCthis"

--- abstract

This document defines metadata objects to support delegating the delivery of
HTTPS content between two or more interconnected CDNs.  Specifically, this
document defines CDNI Metadata interface objects to enable delegation of
of X.509 certificates. This document leverages delegation schemes defined in
RFC9115. RFC 9115 allows delegating entity to remain in full
control of the delegation and be able to revoke the delegation any time and avoids
the need to share private cryptographic key material between the involved entities.

--- middle

# Introduction

Content delivery over HTTPS using two or more CDNs along the path requires
credential management specifically when DNS-based delegation is used.  For
example, an upstream CDN (uCDN) is delegating its credentials to a downstream (dCDN) for content delivery.
This specifically applies when an entity delegates delivery of encrypted
content to another trusted entity.

{{RFC9115}} defines delegation methods where the upstream Content Delivery
Network (uCDN), the holder of the domain, generates on- demand an X.509
certificate for the downstream CDN (dCDN).  For further details, please refer
to {{Section 1 of RFC9115}} and {{Section 5.1.2.1 of RFC9115}}.

This document defines CDNI Metadata to make use of HTTPS delegation between an
uCDN and a dCDN based on the mechanism specified in {{RFC9115}}.  Furthermore,
it includes an addition of a delegation methods to the IANA registry.

{{terminology}} defines terminology used in this document.  {{fci-metadata}}
presents delegation metadata for the FCI interface.  {{mi-metadata}} addresses
the metadata for handling HTTPS delegation with the Metadata Interface.
{{iana}} addresses IANA registry for delegation methods.  {{sec}} covers the
security considerations.

## Terminology {#terminology}

This document uses terminology from CDNI framework documents such as: CDNI
framework document {{RFC7336}}, CDNI requirements {{RFC7337}} and CDNI
interface specifications documents: CDNI Metadata interface {{RFC8006}} and
CDNI Footprint and capabilities {{RFC8008}}.  It also uses terminology from
{{Section 1.1 of RFC8739}}.


# Advertising Delegation Metadata for CDNI through FCI {#fci-metadata}

The Footprint and Capabilities interface as defined in {{RFC8008}}, allows a
dCDN to send a FCI capability type object to a uCDN.

The FCI.Metadata object shall allow a dCDN to advertise the capabilities
regarding the supported delegation methods and their configuration.

The following is an example of the supported delegated methods capability
object for a CDN supporting STAR delegation method.

~~~json
{
  "capabilities": [
    {
      "capability-type": "FCI.Metadata",
      "capability-value": {
        "metadata": [
          "AcmeStarDelegationMethod",
          "... Other supported delegation methods ..."
        ]
      },
      "footprints": [
        "Footprint objects"
      ]
    }
  ]
}
~~~

# ACME Delegation Metadata for CDNI {#mi-metadata}

This section defines the AcmeStarDelegationMethod object which describes
metadata related to the use of ACME/STAR API presented in {{RFC9115}}.

This allows bootstrapping ACME delegation method between a uCDN and a delegate
dCDN.

As expressed in {{RFC9115}}, when an origin has set a delegation to a specific
domain (i.e., dCDN), the dCDN should present to the end-user client a
short-term certificate bound to the master certificate.

~~~aasvg
                                                    .------------.
                            video.cp.example ?     | .-----.      |
                 .---------------------------------->|     |      |
                |                  (a)             | | DNS |  CP  |
                |    .-------------------------------+     |      |
                |   |   CNAME video.ucdn.example   | '-----'      |
                |   |                               '------------'
                |   |
                |   |
    .-----------|---v--.                            .------------.
   |    .-----.-+-----. |   video.ucdn.example ?   | .-----.      |
   |    |     |       +----------------------------->|     |      |
   | UA | TLS |  DNS  | |          (b)             | | DNS | uCDN |
   |    |     |       |<-----------------------------+     |      |
   |    '--+--'-----+-' | CNAME video.dcdn.example | '-----'      |
    '------|----^---|--'                            '------------'
           |    |   |
           |    |   |
           |    |   |                               .------------.
           |    |   |      video.dcdn.example ?    | .-----.      |
           |    |    '------------------------------>|     |      |
           |    |                  (c)             | | DNS |      |
           |     '-----------------------------------+     |      |
           |                   A 192.0.2.1         | +-----+ dCDN |
           |                                       | |     |      |
            '--------------------------------------->| TLS |      |
                        SNI: video.cp.example      | |     |      |
                                                   | '-----'      |
                                                    '------------'
~~~

~~~aasvg
.----.                .----.               .----.                 .----.
|dCDN|                |uCDN|               | CP |                 | CA |
'-+--'                '-+--'               '--+-'                 '--+-'
  |     GET metadata    |                     |                      |
  +--------[CDNI]------>|                     |                      |
  |   200 OK, metadata  |                     |                      |
  |  (inc. dele config) |                     |                      |
  |<-------[CDNI]-------+                     |                      |
  |                     |                     |                      |
  |    GET delegation   |                     |                      |
  +-----[ACME dele]---->|                     |                      |
  | 200 OK, delegation  |                     |                      |
  | (inc. CSR template) |                     |                      |
  |<----[ACME dele]-----+                     |                      |
  |                     |                     |                      |
  +----.                |                     |                      |
  |    |                |                     |                      |
  |  create key pair and|                     |                      |
  |  CSR w/ delegated   |                     |                      |
  |  name               |                     |                      |
  |    |                |                     |                      |
  |<---'                |                     |                      |
  |                     |                     |                      |
  |     POST Order1     |                     |                      |
  +-----[ACME dele]---->|                     |                      |
  |                     |   forward Order1    |                      |
  |                     +-----[ACME dele]---->|                      |
  |                     |                     |     POST Order2      |
  |                     |                     +-----[ACME STAR]----->|
  |                     |                     |                      |
  |                     |                     |<---authorizations--->|
  |                     |                     |                      |
  |<---wait issuance--->|<---wait issuance--->|<---wait issuance---->|
  |                                                                  |
  |              (unauthenticated) GET star-certificate              |
  +----------------------------------------------------------------->|
  |                          certificate #1                          |
  |<-----------------------------------------------------------------+
  |                              ...                                 |
~~~
{: #fig-call-flow artwork-align="center"
   title="Example call-flow of STAR delegation in CDNI showing 2 levels of delegation"}

* Property: acme-delegations

* Description: an array of delegation objects associated with the dCDN account on the uCDN ACME server (see Section 2.3.1 of {{RFC9115}} for the details).

* Type: Objects

* Mandatory-to-Specify: Yes

Below shows both HostMatch and its Metadata related to a host, for example,
here is a HostMatch object referencing "video.example.com" and a list of 2
acme-delegation objects.

Following the example above, the metadata is modeled for
ACMEStarDelegationMethod as follows:

~~~json
{
  "generic-metadata-type": "MI.AcmeStarDelegationMethod",
  "generic-metadata-value": {
    "acme-delegations": [
      "https://acme.ucdn.example/acme/delegation/ogfr8EcolOT",
      "https://acme.ucdn.example/acme/delegation/wSi5Lbb61E4"
    ]
  }
}
~~~

# IANA Considerations {#iana}

This document requests the registration of the following entries under the
"CDNI Payload Types" registry hosted by IANA regarding "CDNI delegation":

| Payload Type | Specification |
|---
| MI.AcmeStarDelegationMethod | {{&SELF}} |

[^to-be-removed]

[^to-be-removed]: RFC Editor: please replace {{&SELF}} with the RFC number of this RFC and remove this note.

## CDNI MI AcmeStarDelegationMethod Payload Type

Purpose:
: The purpose of this Payload Type is to distinguish AcmeStarDelegationMethod
  MI objects (and any associated capability advertisement)

Interface:
: MI

Encoding:
: See {{mi-metadata}}

# Security considerations {#sec}

Delegation metadata proposed here do not alter nor change Security
Considerations as outlined in the following RFCs: An Automatic Certificate
Management Environment (ACME) Profile for Generating Delegated Certificates
{{RFC9115}}; the CDNI Metadata {{RFC8006}} and CDNI Footprint and Capabilities
{{RFC8008}}.

The delegation objects properties such as the list of delegation objects
mentioned in {{mi-metadata}}are critical.  They should be protected by the
proper/mandated encryption and authentication.  Please refer to Sections 7.1,
7.2 and 7.4 of {{RFC9115}}.

--- back

# Acknowledgments
{:unnumbered}

TODO
