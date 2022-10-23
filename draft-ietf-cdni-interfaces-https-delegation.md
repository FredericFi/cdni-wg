---
v: 3

title: CDNI extensions for HTTPS delegation
abbrev: CDNI extensions for HTTPS delegation
[comment]: # title: CDNI metadata for ACME-based delegation
[comment]: # abbrev: CDNI metadata for ACME-based delegation
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
  RFC7975:

entity:
  SELF: "RFCthis"

--- abstract

This document defines metadata objects to support delegating the delivery of
HTTPS content between two or more interconnected CDNs.  Specifically, this
document defines CDNI Metadata interface objects to enable delegation of
X.509 certificates leveraging delegation schemes defined in
RFC9115. RFC 9115 allows delegating entity to remain in full
control of the delegation and be able to revoke it any time and avoids
the need to share private cryptographic key material between the involved entities.

--- middle

# Introduction

Content delivery over HTTPS using two or more cooperating Content Delivery Networks (CDNs) along the path requires
credential management, specifically when DNS-based redirection is used.  In such case an upstream CDN (uCDN) needs to delegate its credentials to a downstream (dCDN) for content delivery.

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

When a uCDN delegates a dCDN to deliver HTTPS traffic using DNS Redirection
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

The ACMEDelegationMethod object allows a uCDN to both define STAR and non-STAR delegation objects depending on the delegation certificate validity.
The ACMEDelegationMethod object is defined with several properties as shown below.

* Property: ACME-delegation

  * Description: a URL pointing at an ACME delegation object, either STAR or non-STAR, associated with the dCDN account on the uCDN ACME server (see {{Section 2.3.1 of RFC9115}} for the details).
  * Type: Source object, according to {{RFC8006}}
  * Mandatory-to-Specify: Yes

* Property: TimeWindow

  * Description: Validity period of the certificate. According to {{RFC8006}}, TimeWindow is defined by defining "start" time of the window, and "end" time of the window. In case of STAR method, the "start" and "end" properties of the window must be understood respectively as the start-date and end-date of the certificate validity. In case of non-STAR method, the "start" and "end" properties of the window must be understood respectively as the notBefore and notAfter fields of the certificate.
  * Type: TimeWindow
  * Mandatory-to-Specify: Yes

In the case that the delegation is STAR-based, the following properties are mandatory to specify:

* Property: Lifetime

  * Description: See {{Section 3.1.1 of RFC8739}}
  * Type: Time, see {{RFC8006}}
  * Mandatory-to-Specify: Yes, only if a STAR delegation method is specified

* Property: Lifetime-adjust

  * Description: See {{Section 3.1.1 of RFC8739}}
  * Type: Time
  * Mandatory-to-Specify: Yes, only if a STAR delegation method is specified

## Examples {#examples}

The following example shows an `ACMEDelegationMethod` object for a STAR-based
ACME delegation.

~~~json
{
  "generic-metadata-type": "MI.ACMEDelegationMethod",
  "generic-metadata-value": {
    "ACME-delegation": "https://acme.ucdn.example/delegation/ogfr",
    "TimeWindow": {
      "start": "2022-10-10T00:00:00Z",
      "end": "2022-10-13T00:00:00Z"
    },
    "Lifetime": 345600,
    "Lifetime-adjust": 259200
  }
}
~~~

The example below shows an `ACMEDelegationMethod` object for a non-STAR ACME
delegation.

~~~json
{
  "generic-metadata-type": "MI.ACMEDelegationMethod",
  "generic-metadata-value": {
    "ACME-delegation": "https://acme.ucdn.example/delegation/wSi5",
    "TimeWindow": {
      "start": "2019-01-10T00:00:00Z",
      "end": "2023-01-20T00:00:00Z"
    }
  }
}
~~~

The following is a complete example showing how a HostMatch {{RFC8006}} and its Metadata
related to a host hold associated delegation metadata.

* HostMatch:

~~~json
{
  "host": "video.example.com",
  "host-metadata": {
    "type": "MI.HostMetadata",
    "href": "https://metadata.ucdn.example/host1234"
  }
}
~~~

* HostMetadata:

~~~json
{
  "paths": "/video",
  "metadata": [ // defining here a STAR delegation
    {
      "generic-metadata-type": "MI.ACMEDelegationMethod",
      "generic-metadata-value": {
        "ACME-delegation": "https://acme.ucdn.example/delegation/wSi5",
        "TimeWindow": {
          "start": "2019-01-10T00:00:00Z",
          "end": "2023-01-20T00:00:00Z"
        }
      }
    }
  ]
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

We would like to thank authors of the {{RFC9115}}, Antonio Augustin Pastor Perales, Diego Lopez, Thomas Fossati and Yaron Sheffer. Additionally, our gratitude to Thomas Fossati who participated in the drafting, reviewing and giving his feedback in finalizing this document. We also thank CDNI, co-chair Kevin Ma for his continual review and feedback during the development of this document.

