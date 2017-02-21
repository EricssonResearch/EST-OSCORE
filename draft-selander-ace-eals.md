---
title: Enrollment with Application Layer Security
# abbrev: EALS
docname: draft-selander-ace-eals-latest

# date: 2017-03-14


# stand_alone: true

ipr: trust200902
area: Security
wg: ACE Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:    # can use array (if all yes) or hash here
#  - toc
#  - sortrefs
#  - symrefs
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: G. Selander
        name: Goeran Selander
        org: Ericsson AB
        street: Farogatan 6
        city: Kista
        code: SE-16480 Stockholm
        country: Sweden
        email: goran.selander@ericsson.com
      -
        ins: S. Raza
        name: Shahid Raza
        org: RISE SICS
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: shahid@sics.se


normative:

  RFC2119:
  RFC7049:
  RFC7252:
  I-D.ietf-cose-msg:
  I-D.ietf-core-object-security:
  I-D.selander-ace-cose-ecdhe:

informative:

  RFC7228:
  RFC7030:
  I-D.richardson-6tisch-dtsecurity-secure-join:
  I-D.ietf-anima-bootstrapping-keyinfra:
  I-D.ietf-6tisch-minimal-security:
  I-D.hartke-core-e2e-security-reqs:
  I-D.ietf-core-coap-tcp-tls:
  I-D.bormann-6lo-coap-802-15-ie:

  

--- abstract


This document specifies public key certificate enrollment procedures authenticated with application layer security protocols suitable for Internet of Things deployments. The protocols leverage existing standards including Constrained Appliction Protocol (CoAP), Consise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format.

--- middle

# Introduction #       {#intro}


Security at the application layer provides an attractive option for protecting Internet of Things (IoT) deployments, in particular in constrained environments {{RFC7228}} or when using CoAP {{RFC7252}}; for example where transport layer security is not sufficient {{I-D.hartke-core-e2e-security-reqs}}, or where it is beneficial that the security protocol is independent of lower layers, such as when securing CoAP over mixed transport protocols.

Application layer security protocols suitable for constrained devices are in development, including the secure communication protocol OSCOAP {{I-D.ietf-core-object-security}}, and the key exchange protocol EDHOC {{I-D.selander-ace-cose-ecdhe}}. OSCOAP defines an extension to the Constrained Application Protocol (CoAP) providing encryption, integrity and replay protection end-to-end between CoAP client and server based on a shared secret. EDHOC is a key establishment protocol providing mutual authentication of client and server based on pre-shared keys, raw public keys or public key certificates, and establishes a shared secret with forward secrecy which may be used with OSCOAP. OSCOAP and EDHOC are built upon other constrained device primitives developed in the IETF: CoAP, CBOR {{RFC7049}} and COSE {{I-D.ietf-cose-msg}}, and makes only a small additional implementation footprint.

Lately there has been a discussion in several IETF working groups about certificate enrollment protocols suitable for IoT devices, to enable zero-touch joining of a device in a network domain.  BRSKI {{I-D.ietf-anima-bootstrapping-keyinfra}} specifies an automated bootstrapping of a remote secure key infrastructure (BRSKI) using vendor installed X.509 certificate, in combination with a vendor authorized service on the Internet. BRSKI is referencing Enrolment over Secure Transport (EST) {{RFC7030}}. 

This document describes an efficient join/certificate enrollment procedure called Enrollment with Application Layer Security (EALS), using the aforementioned application layer security protocols. For devices already implementing OSCOAP or EDHOC, the enrollment protocols re-use much of the code base and is only a minor addition.

The problem statement from BRSKI is imported into this document. The limitations of applicability to energy constrained devices due to credential size applies also to this document, and further work is needed to specify certificate formats relevant to constrained devices. Having said that, one rationale for this document is a more optimized message exchange, which is favorable in low-power deployments. Related work include {{I-D.richardson-6tisch-dtsecurity-secure-join}} which addresses the low-power problem statement, and {{I-D.ietf-6tisch-minimal-security}} which describes a one-touch procedure using OSCOAP and EDHOC.



## Terminology ##  {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}. These
words may also appear in this document in lowercase, absent their
normative meanings.

Terminology taken from \[ \]

Pledge:
:  the prospective device, which has the identity provided to
   at the factory.  Neither the device nor the network knows if the
   device yet knows if this device belongs with this network.

Joined Node:
: the prospective device, after having completing the join process, often
  just called a Node.

Join Proxy (JP):
:  a stateless relay that provides connectivity between the pledge
   and the join registrar/coordinator.

Join Registrar/Coordinator (JRC):
:  a central entity responsible for authentication and authorization of joining
   nodes.

# Enrollment Protocols # {#protocols}

## Overview ##

Two different certificate enrollment protocols are described in this document; the simple (EALS) enrollment protocol and the full (EALS) enrollment protocol. The Pledge is EALS client and the Join Registrar/Coordinator is EALS server.

The simple enrollment protocol is a two-pass protocol: the EALS client sends a certificate enrollment request and, if granted, the EALS server responds with a certificate. The protocol is protected with OSCOAP and assumes that both EALS client and EALS server implement CoAP. (The CoAP messages may, however, be carried using various transport protocols e.g. UDP, TCP {{I-D.ietf-core-coap-tcp-tls}}, 802.15.4 IE {{I-D.bormann-6lo-coap-802-15-ie}}).

The full enrollment protocol is preceding the simple enrolment protocol with a mutual authentication, authorization and key establishment procedure embedded in the 3-pass EDHOC protocol. The full enrollment protocol can be transported over any channel, and the binding to CoAP is described in this document.



## Simple Enrollment ## {#simple}

The simple enrollment is a 2-pass protocol between EALS client and EALS server. Simple enrollment assumes that client and server has pre-established a shared master secret associated to the identity which the EALS client requesting to enroll. 

The simple enrollment protocol may be used for re-enrollment e.g. if a previous full enrollment procedure has been sucessfully performed.

~~~~~~~~~~~

 EALS                                                   EALS 
client                                                 server

  |                                                       | 
  | POST /eals        (ObjectSecurity; Payload: PKCS#10)  |  
  +------------------------------------------------------>|
  |                                                       |          
  |  2.04 Changed   (ObjectSecurity; Payload: PKCS#7)     |
  |<------------------------------------------------------+
  |                                                       |       

~~~~~~~~~~~
{: #simple-enroll title="The Simple EALS Protocol."}
{: artwork-align="center"}


The protocol consists of an OSCOAP exchange using the security context derived from the shared master secret. The EALS client sends an OSCOAP POST to /simpleeals, request payload is the certificate signing request (PKCS#10). The EALS server MUST verify that the DN in PKCS#10 corresponds to the security context with which the received OSCOAP request was protected. Since OSCOAP has sequence number-based replay protection, the EALS server rejects old requests. 

If the request is successful, the EALS server responds with 2.04 Changed and returns a public key certificate (PKCS#7) matching the requested PKCS#10 to the EALS client.

The protocol is entirely embedded in CoAP and is protected end-to-end when sent over any transport and via intermediary nodes. One candidate embedding of EALS into a bootstrapping architecture is as described in {{I-D.ietf-6tisch-minimal-security}}  where the Plegde is EALS/CoAP client, the Join Registrar/Coordinator is EALS/CoAP server and Join Proxy is a CoAP proxy.

TBD Details

TBD Stateless proxy, recap and reference minimal security draft


## Full Enrollment ##


The full enrollment procedure consists of a mutual authentication and authorization step, using EDHOC with an enrollment extension defined in this section, followed by the simple enrollment protocol ({{simple}}). The EDHOC protocol {{I-D.selander-ace-cose-ecdhe}} establishes the master secret used with the simple enrollment protocol for first enrollment or subsequent re-enrollments.

EDHOC is a 3-pass client-server key exchange protocol providing mutual authentication between client and server, allowing application specific extensions. The parties may authenticate using pre-established keys, raw public keys or public key certificates. The focus here is on the last option, where the pre-established certificates are by issued the manufacturer, as described in BRSKI {{I-D.ietf-anima-bootstrapping-keyinfra}}.

The EDHOC protocol is exended with an optional audit/ownership voucher {{I-D.ietf-anima-bootstrapping-keyinfra}}. The voucher is retrieved by the EALS server using mechanism out of scope for this document.  server {{I-D.ietf-anima-bootstrapping-keyinfra}}. The voucher is transported by leveraging the extensions mechanism built-in into EDHOC. The extension in EDHOC message_3 is encrypted by the EDHOC protocol.



~~~~~~~~~~~

 EALS                                                   EALS 
client                                                 server

  |                                                       |
  |                                                       |
  +------------------------------------------------------>|
  |                                                       |
  |                    EDHOC message_1                    |
  |<------------------------------------------------------+
  |                                                       |
  |                    EDHOC message_2                    |
  +------------------------------------------------------>|
  |                                                       |
  |             EDHOC message_3  (EXT_3 = Voucher)        |
  |<------------------------------------------------------+
  |                                                       |    

~~~~~~~~~~~
{: #full-enroll title="Mutual authentication and authorization."}
{: artwork-align="center"}


The enrollment procedure described here is assuming that the EALS server/Join Registrar/Coordinator is EDHOC client, see figure {{full-enroll}}. This setting has the properties that the 

1. The EALS server initiates the protocol
2. The EALS client is authenticed first (EDHOC message_2)

Item 1. allows the EALS server to orchestrate many concurrent enrollments. Item 2. allows the the EALS server to authenticate and authorize the EALS client before completing the protocol.

For certain deployment settings the reverse roles may be favorable, and it is straightforward to embed the enrolment protocol in EDHOC with interchanged roles. The details are FFS.



TBD Detail the protocol

TBD CoAP binding. Same as in EDHOC?

TBD name of resource? POST /fulleals?

TBD CoAP Response codes to communicate success or failure of the EALS function?



# Security Considerations # {#sec-cons}



# Privacy Considerations #

TODO


# IANA Considerations # {#iana}



# Acknowledgments #

The authors wants to thank  

--- back

# Examples {#examples}


--- fluff

