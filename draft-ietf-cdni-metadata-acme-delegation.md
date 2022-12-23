---
v: 3

title: CDNI metadata for ACME-based delegation
abbrev: CDNI metadata for ACME-based delegation
docname: draft-ietf-cdni-interfaces-https-delegation-latest

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
  RFC7337:
  RFC7975:

entity:
  SELF: "RFCthis"

--- abstract

This document defines metadata objects to support delegating the delivery of
HTTPS content between two or more interconnected CDNs.  Specifically, this
document defines CDNI Metadata interface objects to enable delegation of
X.509 certificates leveraging delegation schemes defined in
RFC9115. RFC9115 allows delegating entities to remain in full
control of the delegation and be able to revoke it any time and avoids
the need to share private cryptographic key material between the involved entities.

--- middle

# Introduction

Content delivery over HTTPS using two or more cooperating Content Delivery Networks (CDNs) along the path requires
credential management, specifically when DNS-based redirection is used.  In such cases, an upstream CDN (uCDN) needs to delegate its credentials to a downstream (dCDN) for content delivery.

{{RFC9115}} defines delegation methods that allow a uCDN on behalf of the content provider, the holder of the domain, to generate on-demand an X.509
certificate that binds the designated domain name with a key-pair owned by the dCDN.  For further details, please refer
to {{Section 1 of RFC9115}} and {{Section 5.1.2.1 of RFC9115}}.

This document defines CDNI Metadata to make use of HTTPS delegation between a
uCDN and a dCDN based on the mechanism specified in {{RFC9115}}.  Furthermore,
it adds a delegation method to the "CDNI Payload Types" IANA registry.

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

The Footprint and Capabilities interface defined in {{RFC8008}} allows a
dCDN to send a FCI capability type object to a uCDN.

The FCI.Metadata object allows a dCDN to advertise the capabilities
regarding the supported delegation methods and their configuration.

The following is an example of the supported delegated methods capability
object for a dCDN implementing the ACME delegation method.

~~~json
{
  "capabilities": [
    {
      "capability-type": "FCI.Metadata",
      "capability-value": {
        "metadata": [
          "ACMEDelegationMethod",
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

When a uCDN delegates to a dCDN to deliver HTTPS traffic using DNS Redirection
{{RFC7975}}, the dCDN must use a certificate bound to the origin's name to
successfully authenticate to the end-user (see also {{Section 5.1.2.1 of
RFC9115}}).

To that end, this section defines the AcmeDelegationMethod object which
describes metadata for using the ACME delegation interface {{RFC9115}}.

The ACMEDelegationMethod applies to both ACME STAR delegation, which provides a
delegation model based on short-term certificates with automatic renewal {{Section 2.3.2 of RFC9115}}, and
non-STAR delegation, which allows delegation between CDNs using long-term
certificates {{Section 2.3.3 of RFC9115}}.

{{fig-call-flow}} provides a high-level view of the combined CDNI and ACME
delegation message flows to obtain STAR certificate bound to the origin's name.

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

{{acmedeleobj}} defines the objects used for bootstrapping the ACME delegation
method between a uCDN and a delegate dCDN.

## ACMEDelegationMethod Object {#acmedeleobj}

The ACMEDelegationMethod object allows a uCDN to both define STAR and non-STAR delegation depending on the delegation certificate validity.
The ACMEDelegationMethod object is defined with the properties shown below.

* Property: acme-delegation

  * Description: a URL pointing at an ACME delegation object, either STAR or non-STAR, associated with the dCDN account on the uCDN ACME server (see {{Section 2.3.1.3 of RFC9115}} for the details).
  * Type: String
  * Mandatory-to-Specify: Yes

* Property: time-window

  * Description: Validity period of the certificate. According to {{Section 4.3.4 of RFC8006}}, a TimeWindow object is defined by a window "start" time, and a window "end" time. In case of STAR method, the "start" and "end" properties of the window must be understood respectively as the start-date and end-date of the certificate validity. In the case of the non-STAR method, the "start" and "end" properties of the window must be understood respectively as the notBefore and notAfter fields of the certificate.
  * Type: TimeWindow
  * Mandatory-to-Specify: Yes

* Property: lifetime

  * Description: See {{Section 3.1.1 of RFC8739}}
  * Type: Integer
  * Mandatory-to-Specify: Yes, only if a STAR delegation method is specified

* Property: lifetime-adjust

  * Description: See {{Section 3.1.1 of RFC8739}}
  * Type: Integer
  * Mandatory-to-Specify: Yes, only if a STAR delegation method is specified

### Examples

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
: See {{mi-metadata}}

# Security Considerations {#sec}

The metadata object defined in this document does not introduce any new
security or privacy concerns over those already discussed in {{RFC9115}},
{{RFC8006}} and {{RFC8008}}.

The reader is expected to understand the ACME delegation trust model ({{Section
7.1 of RFC9115}}) and security goal ({{Section 7.2 of RFC9115}}), in particular
the criticality around the protection of the user account associated with the
delegation.
For example, the delegation object pointed by the `acme-delegation` property is
only accessible to the holder of the account key, which is allowed to fetch its
content exclusively via POST-as-GET ({{Section 2.3.1.2 of RFC9115}}).

In addition, the Metadata interface authentication and confidentiality requirements
defined in {{Section 8.1. of RFC80006}}, {{Section 8.2. of RFC8006}} and {{Section 8.3. of RFC8006}}
MUST be followed to prevent unauthorized access to the ACME delegation object.

Implementers MUST adhere to the security considerations defined in the CDNI Request Routing: Footprint and Capabilities Semantics, {{Section 7. of RFC8008}}.

When TLS is used to achieve the above security objectives, the general TLS
usage guidance in {{RFC9325}} MUST be followed.

--- back

# Acknowledgments
{:unnumbered}

We would like to thank authors of the {{RFC9115}}, Antonio Augustin Pastor Perales, Diego Lopez, Thomas Fossati and Yaron Sheffer. Additionally, our gratitude to Thomas Fossati who participated in the drafting, reviewing and giving his feedback in finalizing this document. We also thank CDNI co-chair Kevin Ma for his continual review and feedback during the development of this document.

