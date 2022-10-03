---
v: 4

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

The Footprint and Capabilities interface as defined in {{RFC8008}}, allows a
dCDN to send a FCI capability type object to a uCDN.

The FCI.Metadata object shall allow a dCDN to advertise the capabilities
regarding the supported delegation methods and their configuration.

The following is an example of the supported delegated methods capability
object for a CDN supporting ACME delegation method.

~~~json
{
  "capabilities": [
    {
      "capability-type": "FCI.Metadata",
      "capability-value": {
        "metadata": [
          "AcmeDelegationMethod",
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

This section defines the AcmeDelegationMethod object which describes
metadata related to the use of ACME API presented in {{RFC9115}}.

This section applies to ACME/STAR delegation, which allows short-term certificate delegation method and its automatic certificate renewal. It applies as well to non-STAR delegation method, which allows delegation between CDNs with longer term certificate.

The following objects shall allow bootstrapping ACME delegation method, both for STAR and non-STAR approaches, between a uCDN and a delegate dCDN.

As expressed in {{RFC9115}}, when an origin has set a delegation to a specific
domain (i.e., dCDN), the dCDN should present to the end-user client a
short-term certificate bound to the master certificate.

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

## AcmeDelegationMethod object

The AcmeDelegationMethod object contains a source to ACME delegation method object, either STAR or non-STAR based, as defined in {{RFC9115}}, as well as certificate validity in order to indicate the renewal periodicity.

The following properties are defined:

* Property: Acme-delegation

* Description: a URL pointing at delegation objects associated with the dCDN account on the uCDN ACME server (see Section 2.3.1 of {{RFC9115}} for the details).

* Type: Source object

* Mandatory-to-Specify: Yes


* Property: TimeWindow

* Description: Validity period of the certificate. ACcording to {{RFC8006}}, TimeWindow is defined by defining "start" time of the window, and "end" time of the window. In case of STAR method, the "start" and "end" properties of the window must be understood respectively as the start-date and end-date of the certificate validity. In case of non-star method, the "start" and "end" properties of the window must be understood respectively as the "not-before" and "not-after" fields of the certificate.

* Type: TimeWindow

* Mandatory-to-Specify: Yes


In the case of a STAR-method, the following properties are mandatory to specify.


* Property: STAR-method

* Description: boolean that specifies a STAR-method

* Type: Boolean

* Mandatory-to-Specify: Yes for STAR delegation method


* Property: Lifetime

* Description: See {{Section 3.1.1 of RFC8739}}

* Type: Time

* Mandatory-to-Specify: Yes for STAR delegation method


* Property: Lifetime-adjust

* Description: See {{Section 3.1.1 of RFC8739}}

* Type: Time

* Mandatory-to-Specify: Yes for STAR-delegation method



## Example

Below shows both HostMatch and its Metadata related to a host, for example,
here is a HostMatch object referencing "video.example.com" and a list of 2
acme-delegation objects.

Following the example above, the metadata is modeled for
ACMEDelegationMethod as follows:

~~~json
{
  "generic-metadata-type": "MI.AcmeDelegationMethod",
  "generic-metadata-value": [
     {
        "Acme-delegation": "https://acme.ucdn.example/acme/delegation/ogfr8EcolOT",
        "TimeWindow": {
             "start": "2019-01-10T00:00:00Z",
             "end": "2019-01-20T00:00:00Z"
        },
        "Lifetime": 345600, // 4 days
        "Lifetime-adjust": 259200, // 3 days
        "STAR-method": true
     },
     {
       "Acme-delegation": "https://acme.ucdn.example/acme/delegation/wSi5Lbb61E4",
       "TimeWindow": {
            "start": "2019-01-10T00:00:00Z",
            "end": "2019-01-20T00:00:00Z"
       }
     }
  ]
}

~~~



# IANA Considerations {#iana}

This document requests the registration of the following entries under the
"CDNI Payload Types" registry hosted by IANA regarding "CDNI delegation":

| Payload Type | Specification |
|---
| MI.AcmeDelegationMethod | {{&SELF}} |

[^to-be-removed]

[^to-be-removed]: RFC Editor: please replace {{&SELF}} with the RFC number of this RFC and remove this note.

## CDNI MI AcmeDelegationMethod Payload Type

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

The authors of this document wishes to thank all the authors of the {{RFC9115}} and more specifically, Thomas Fossati who has participated in the drafting, reviewing and discussion of this draft.

