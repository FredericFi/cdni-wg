---
v: 3

title: CDNI delegation using Automated Certificate Management Environment
abbrev: CDNI delegation using ACME
docname: draft-ietf-cdni-delegation-acme-latest

category: std
consensus: true
submissiontype: IETF

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Content Delivery Networks Interconnection"
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
  group: "Content Delivery Networks Interconnection"
  mail: "cdni@ietf.org"
  github: "FredericFi/cdni-wg"

normative:
  RFC8006:
  RFC8008:
  RFC8739:
  RFC9115:
  RFC9325:

informative:
  RFC7336:

entity:
  SELF: "RFCthis"

--- abstract

This document defines metadata to support delegating the delivery of
HTTPS content between two or more interconnected Content Delivery Networks (CDNs).  Specifically, this
document defines a Content Delivery Network Interconnection (CDNI) Metadata interface object to enable delegation of
X.509 certificates leveraging delegation schemes defined in
RFC9115. Per RFC 9115, delegating entities can remain in full control of the delegation and can revoke it at any time. This avoids the need to share private cryptographic key material between the involved entities.


--- middle

# Introduction

Content delivery over HTTPS using two or more cooperating CDNs along the path requires
credential management, specifically when DNS-based redirection is used.  In such cases, an upstream CDN (uCDN) needs to delegate its credentials to a downstream (dCDN) for content delivery.

{{RFC9115}} defines delegation methods that allow a uCDN on behalf of the content provider, the holder of the domain, to generate on-demand an X.509
certificate that binds the designated domain name with a key-pair owned by the dCDN.  For further details, please refer
to {{Section 1 of RFC9115}} and {{Section 5.1.2.1 of RFC9115}}.

This document defines CDNI Metadata to make use of HTTPS delegation between a
uCDN and a dCDN based on the mechanism specified in {{RFC9115}}.  Furthermore,
it adds a delegation method to the "CDNI Payload Types" IANA registry.

{{fci-metadata}} presents delegation metadata for the Footprint & Capabilities Advertisement interface (FCI).  {{mi-metadata}} addresses the metadata for handling HTTPS delegation with the Metadata Interface.


## Terminology {#terminology}

This document uses terminology from CDNI framework documents such as: CDNI
framework document {{RFC7336}} and CDNI
interface specifications documents: CDNI Metadata interface {{RFC8006}} and
CDNI Footprint and Capabilities Advertisement interface {{RFC8008}}.  It also uses terminology from
{{Section 1.2 of RFC8739}} and {{Section 1.1 of RFC9115}}, including Short-Term, Automatically Renewed (STAR), as applied to X.509 certificates.

{::boilerplate bcp14}

# Advertising Delegation Metadata for CDNI through FCI {#fci-metadata}

The Footprint and Capabilities Advertisement interface (FCI) defined in {{RFC8008}} allows a
dCDN to send a FCI capability type object to a uCDN.

This document uses the CDNI Metadata capability object serialization from {{RFC8008}} for a CDN that supports
delegation methods.

The following is an example of the supported delegated methods capability
object for a dCDN implementing the ACME delegation method.

~~~json
{
  "capabilities": [
    {
      "capability-type": "FCI.Metadata",
      "capability-value": {
        "metadata": [
          // list of supported delegation methods
          "ACMEDelegationMethod"
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

When a uCDN delegates the delivery of HTTPS traffic to a dCDN using DNS Redirection
{{RFC7336}}, the dCDN must use a certificate bound to the origin's name to
successfully authenticate to the end-user (see also {{Section 5.1.2.1 of
RFC9115}}).

To that end, this section defines the AcmeDelegationMethod object which
describes metadata for using the ACME delegation interface {{RFC9115}}.

The ACMEDelegationMethod applies to both ACME STAR delegation, which provides a
delegation model based on short-term certificates with automatic renewal ({{Section 2.3.2 of RFC9115}}), and
non-STAR delegation, which allows delegation between CDNs using long-term
certificates ({{Section 2.3.3 of RFC9115}}).

{{fig-call-flow}} provides a high-level view of the combined CDNI and ACME
delegation message flows to obtain STAR certificate from the Certificate Authority (CA) bound to the Content Provider's (CP) name.

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

{:aside}
> Note: The delegation object defined in {{Section 2.3.1.3 of RFC9115}} only allows DNS mappings to be specified using CNAME RRs.  A future document updating {{RFC9115}} could expand the delegation object to also include SVCB/HTTPS-based {{?RFC9460}} mappings.

{{acmedeleobj}} defines the objects used for bootstrapping the ACME delegation
method between a uCDN and a delegate dCDN.

## ACMEDelegationMethod Object {#acmedeleobj}

The ACMEDelegationMethod object allows a uCDN to define both STAR and non-STAR delegation. The dCDN, the consumer of the delegation, can determine the type of delegation by the presence (or absence) of the "lifetime" property. That is, the presence of the "lifetime" property explicitly means a short-term delegation with lifetime of the certificate based on that property (and the optional "lifetime-adjust" attribute). A non-STAR delegation will not have the "lifetime" property in the delegation.  See also the examples in {{examples}}.

The ACMEDelegationMethod object is defined with the properties shown below.

* Property: acme-delegation

  * Description: a URL pointing at an ACME delegation object, either STAR or non-STAR, associated with the dCDN account on the uCDN ACME server (see {{Section 2.3.1.3 of RFC9115}} for the details). The URL MUST use the https scheme.
  * Type: String
  * Mandatory-to-Specify: Yes

* Property: time-window

  * Description: Validity period of the certificate. According to {{Section 4.3.4 of RFC8006}}, a TimeWindow object is defined by a window "start" time, and a window "end" time. In case of STAR method, the "start" and "end" properties of the window MUST be understood respectively as the start-date and end-date of the certificate validity. In the case of the non-STAR method, the "start" and "end" properties of the window MUST be understood respectively as the notBefore and notAfter fields of the certificate.
  * Type: TimeWindow
  * Mandatory-to-Specify: Yes

* Property: lifetime

  * Description: See lifetime in {{Section 3.1.1 of RFC8739}}
  * Type: Integer
  * Mandatory-to-Specify: Yes, only if a STAR delegation method is specified

* Property: lifetime-adjust

  * Description: See lifetime-adjust in {{Section 3.1.1 of RFC8739}}
  * Type: Integer
  * Mandatory-to-Specify: No

### Examples {#examples}

The following example shows an `ACMEDelegationMethod` object for a STAR-based
ACME delegation.

~~~json
{
  "generic-metadata-type": "MI.ACMEDelegationMethod",
  "generic-metadata-value": {
    "acme-delegation": "https://acme.ucdn.example/delegation/ogfr",
    "time-window": {
      "start": 1665417434,
      "end": 1665676634
    },
    "lifetime": 345600,
    "lifetime-adjust": 259200
  }
}
~~~

The example below shows an `ACMEDelegationMethod` object for a non-STAR ACME
delegation. The delegation object is defined as per {{Section 4.3 of RFC8006}}.

~~~json
{
  "generic-metadata-type": "MI.ACMEDelegationMethod",
  "generic-metadata-value": {
    "acme-delegation": "https://acme.ucdn.example/delegation/wSi5",
    "time-window": {
      "start": 1570982234,
      "end": 1665417434
    }
  }
}
~~~

# IANA Considerations {#iana}

This document requests the registration of the following entry under the
"CDNI Payload Types" registry:

| Payload Type | Specification |
|---
| MI.ACMEDelegationMethod | {{&SELF}} |

[^to-be-removed]

[^to-be-removed]: RFC Editor: please replace {{&SELF}} with the RFC number of this RFC and remove this note.

## CDNI MI ACMEDelegationMethod Payload Type

Purpose:
: The purpose of this Payload Type is to distinguish AcmeDelegationMethod
  MI objects (and any associated capability advertisement)

Interface:
: MI/FCI

Encoding:
: See {{acmedeleobj}}

# Security Considerations {#sec}

The metadata object defined in this document does not introduce any new
security or privacy concerns over those already discussed in {{RFC9115}},
{{RFC8006}} and {{RFC8008}}.

The reader is expected to understand the ACME delegation trust model ({{Section
7.1 of RFC9115}}) and security goal ({{Section 7.2 of RFC9115}}).]). In particular, the reader is expected to understand the criticality of the protection of the user account associated with the delegation; this account authorizes all the security-relevant operations between a dCDN and a uCDN over the ACME channel.

The dCDN's ACME account is also relevant to the privacy of the entire scheme;
for example, the `acme-delegation` resource in the Metadata object is only
accessible to the holder of the account key, who is allowed to fetch its
content exclusively via POST-as-GET ({{Section 2.3.1.2 of RFC9115}}).

In addition, the Metadata interface authentication and confidentiality
requirements defined in {{Section 8 of RFC8006}} MUST be followed.

Implementers MUST adhere to the security considerations defined in the CDNI Request Routing: Footprint and Capabilities Semantics, {{Section 7 of RFC8008}}.

When TLS is used to achieve the above security objectives, the general TLS
usage guidance in {{RFC9325}} MUST be followed.

--- back

# Acknowledgments
{:unnumbered}

We would like to thank authors of the {{RFC9115}}, Antonio Augustin Pastor Perales, Diego Lopez, Thomas Fossati and Yaron Sheffer. Additionally, our gratitude to Thomas Fossati who participated in the drafting, reviewing and giving his feedback in finalizing this document. We also thank CDNI co-chair Kevin Ma for his continual review and feedback during the development of this document.

