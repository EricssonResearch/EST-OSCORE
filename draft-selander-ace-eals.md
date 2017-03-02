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

  RFC5272:
  RFC7228:
  RFC7030:
  I-D.richardson-6tisch-dtsecurity-secure-join:
  I-D.ietf-anima-bootstrapping-keyinfra:
  I-D.ietf-6tisch-minimal-security:
  I-D.hartke-core-e2e-security-reqs:
  I-D.ietf-core-coap-tcp-tls:
  I-D.bormann-6lo-coap-802-15-ie:
  I-D.ietf-ace-oauth-authz:
  I-D.seitz-ace-oscoap-profile:
  

--- abstract


This document specifies public key certificate enrollment procedures authenticated with application layer security protocols suitable for Internet of Things deployments. The protocols leverage existing IoT standards including Constrained Appliction Protocol (CoAP), Consise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format.

--- middle

# Introduction #       {#intro}


Security at the application layer provides an attractive option for protecting Internet of Things (IoT) deployments, in particular in constrained environments {{RFC7228}} and when using CoAP {{RFC7252}}; for example where transport layer security is not sufficient {{I-D.hartke-core-e2e-security-reqs}}, or where it is beneficial that the security protocol is independent of lower layers, such as when securing CoAP over mixed transport protocols.

Application layer security protocols suitable for constrained devices are in development, including the secure communication protocol OSCOAP {{I-D.ietf-core-object-security}}. OSCOAP defines an extension to the Constrained Application Protocol (CoAP) providing encryption, integrity and replay protection end-to-end between CoAP client and server based on a shared secret. The shared secret can be established in different ways e.g. using a trusted third party such as in ACE {{I-D.ietf-ace-oauth-authz}}, or using a key exchange protocol such as EDHOC {{I-D.selander-ace-cose-ecdhe}}. OSCOAP and EDHOC are built upon other constrained device primitives developed in the IETF: CoAP, CBOR {{RFC7049}} and COSE {{I-D.ietf-cose-msg}}, and makes only a small additional implementation footprint.

Lately, there has been a discussion in several IETF working groups about certificate enrollment protocols suitable for IoT devices, to support the use case of an IoT device joining a new network domain and establishing credentials valid in this domain. This document describes Enrollment with Application Layer Security (EALS), a certificate enrollment procedure based on the Simple PKI Requests and Responses of CMC {{RFC5272}}. EALS uses OSCOAP as a channel for the enrollment protocol, and describes how ACE and EDHOC can be used for establishing an authenticated and authorized channel.

This work is inspired by the Enrollment over Secure Transport (EST) protocol {{RFC7030}}, which also is based on CMC, but EALS is secured on application layer instead of on transport layer.


## Terminology ##  {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}. These
words may also appear in this document in lowercase, absent their
normative meanings.


# Simple Enrollment # {#simple-enroll}

This section describes the simple enrollment protocol which is an embedding of the Simple PKI Request/Response protocol of CMC {{RFC5272}} in Object Secure CoAP (OSCOAP) {{I-D.ietf-core-object-security}}. 

The simple enrollment protocol is a 2-pass protocol between EALS client and EALS server, see {{fig-simple-enroll}}. The protocol assumes that both EALS client and EALS server implement CoAP, and the Object-Security option of CoAP (OSCOAP). 


~~~~~~~~~~~

 EALS                                                   EALS 
client                                                 server

  |                                                       | 
  | POST /eals      (Object-Security; Payload: PKCS #10)  |  
  +------------------------------------------------------>|
  |                                                       |          
  |  2.04 Changed   (Object-Security; Payload: PKCS #7)   |
  |<------------------------------------------------------+
  |                                                       |       

~~~~~~~~~~~
{: #fig-simple-enroll title="The Simple Enrollment Protocol."}
{: artwork-align="center"}


The simple enrollment protocol consists of a CoAP message exchange. 

The EALS client sends a CoAP request:

 * Method is POST
 * Uri-Path is "eals"
 * Object-Security option is present
 * Payload is the CMC Simple PKI Request {{RFC5272}} (i.e. a PKCS #10 certification request).

 
If successful, the EALS server sends a CoAP response: 

 * Code is 2.04 (Changed)
 * Content-Format is "application/pkcs7-mime" (TBD)
 * Object-Security option is present
 * Payload is a certs-only CMC Simple PKI Response {{RFC5272}} (i.e the issued certificate)


OSCOAP protects the CoAP message exchange between the endpoints over any transport and via intermediary nodes. The OSCOAP protection requires that a security context is shared between client and server, and the security context can be derived from a set of Input Parameters (Section 3.3 of {{I-D.ietf-core-object-security}}):

* Master Secret
* Sender ID
* Recipient ID

The server MUST verify that the Master Secret is associated to the Distinguished Name for which the client is requesting a certificate. Examples of how these input parameters are established in client and server are described in {{establish-master-secret}}.

Note 1: The encodings and formats used in CMC may later be updated with other equivalents more adapted to constrained environments.

Note 2: OSCOAP protects the CoAP message exchange independent of underlying protocol e.g. UDP, TCP {{I-D.ietf-core-coap-tcp-tls}}, or 802.15.4 IE {{I-D.bormann-6lo-coap-802-15-ie}}.

TBD: Further details


# Establishment of Master Secret # {#establish-master-secret}


In order to deploy OSCOAP between EALS client and EALS server, the OSCOAP input parameters needs to established (Section 3.3 of {{I-D.ietf-core-object-security}}), in particular a master secret. In this section we present two application layer protocols for establishing the input parameters: the authorization protocol ACE and the key exchange protocol EDHOC.

## ACE  ##

The ACE protocol framework {{I-D.ietf-ace-oauth-authz}} is an adaptation of OAuth 2.0 to IoT environments. ACE describes different message flows for a Client to get authorized access to a Resource Server (RS) by leveraging an Authorization Server (AS). 

The Token Introspection flow (Section 7 of {{I-D.ietf-ace-oauth-authz}}) allows an RS to access authorization information relating to a client provided access token. If the access token is valid, the RS obtains information about the access rights and key of the client, and also a client token containing the same shared key protected for the legitimate client (Section 7.4 of {{I-D.ietf-ace-oauth-authz}}, {{ACE-introspect}}).

By mapping the EALS client and server to the ACE client and resource server, respectively, this application of ACE enables the authorization of EALS client and establishment of a shared key. The access token is in this case not bound to a particular resource server and could be provisioned to the client during manufacture. Key can be used as master secret with OSCOAP in the simple enrollment protocol in {{simple-enroll}}. The access rights include the right to get enrolled in this key infrastructure.


~~~~~~~~~~~
                     Resource       Authorization
    Client            Server           Server
       |                |                |
       |                |                |
       +--------------->|                |
       |  POST          |                |
       |  Access Token  |                |
       |                +--------------->|
       |                | Introspection  |
       |                |    Request     |
       |                |                |
       |                +<---------------+
       |                | Introspection  |
       |                |   Response     |
       |                | + Client Token |
       |<---------------+                |
       |  2.01 Created  |                |
       | + Client Token |


~~~~~~~~~~~
{: #ACE-introspect title="ACE Token Introspection with Client Token."}
{: artwork-align="center"}


Section 2 of {{I-D.seitz-ace-oscoap-profile}} defines additional common header parameters for COSE_Key structure that are used to carry OSCOAP input parameters Sender and Recipient ID.
OSCOAP master secret is transported as part of the symmetric COSE_Key object. 
This document uses the same construct.
COSE_Key object with OSCOAP input parameters present is transported as part of the introspection response and the client token. 

For the benefit of the client authorizing the enrollment, this document defines an additional common parameter for the client token called voucher, extending the definition in Section 7.4 of {{I-D.ietf-ace-oauth-authz}}:

~~~~~~~~~~~
voucher
    OPTIONAL. Contains authorization information about the server, 
    e.g. ownership voucher. The encoding is TBD. 
~~~~~~~~~~~

~~~~~~~~~~~
/-------------------+----------+-----------------\
| Parameter name    | CBOR Key | Major Type      |
|-------------------+----------+-----------------|
| voucher           | TBD      | 2 (byte string) |
\-------------------+----------+-----------------/
~~~~~~~~~~~
{: #ACE-cbor-mapping-voucher title="CBOR mapping of parameters extending the client token."}
{: artwork-align="center"}



## EDHOC ##

EDHOC {{I-D.selander-ace-cose-ecdhe}} is a key establishment protocol encoded with CBOR and using COSE that may be transported with e.g. CoAP.

EDHOC provides mutual authentication of client and server based on pre-shared keys, raw public keys or public key certificates. EDHOC establishes a shared secret with forward secrecy which may be used by different applications, such as by OSCOAP in the simple enrollment protocol, {{simple-enroll}}.

EDHOC also negotiates the algorithm the application will use with the master secret, and allows application specific extensions and key derivation.



~~~~~~~~~~~

Party U                                             Party V
   |                   E_U, EXT_1                     |
   +------------------------------------------------->|
   |                                                  |
   |      E_V, AEAD(EXT_2, ID_V, Sig(V; E_U, E_V))    |
   |<-------------------------------------------------+
   |                                                  |
   |         AEAD(EXT_3, ID_U, Sig(U; E_V, E_U))      |
   +------------------------------------------------->|
   |                                                  |


~~~~~~~~~~~
{: #fig-EDHOC title="The EDHOC protocol (simplification)."}
{: artwork-align="center"}




To address the authorization aspects the EDHOC protocol is extended with an optional audit/ownership voucher {{I-D.ietf-anima-bootstrapping-keyinfra}}. The voucher is retrieved by the EALS server using mechanism out of scope for this document.  The voucher is transported by leveraging the extensions mechanism built-in into EDHOC. The extension in EDHOC message_3 is encrypted by the EDHOC protocol.


~~~~~~~~~~~
 EALS                                     EALS 
client                                   server

  |                                        |                       
  |                                        |                       
  +--------------------------------------->|                       
  |                                        |                       
  |             EDHOC message_1            |                       
  |<---------------------------------------+                       
  |                                        |                       
  |    EDHOC message_2   (EXT_2 = Nonce)   |                       
  +--------------------------------------->|    Third party           
  |                                        | <----------------->            
  |    EDHOC message_3  (EXT_3 = Voucher)  |    authorization
  |<---------------------------------------+                       
  |                                        |                        

~~~~~~~~~~~
{: #fig-2 title="EDHOC in EALS."}
{: artwork-align="center"}


The enrollment procedure described here is assuming that the EALS server is EDHOC client, see {{fig-2}}. This setting has the properties that the 

1. The EALS server initiates the protocol
2. The EALS client is authenticated first (EDHOC message_2)

Item 1. allows the EALS server to orchestrate many concurrent enrollments. Item 2. allows the EALS server to authenticate and authorize the EALS client before completing the protocol.

For certain deployment settings the reverse roles may be favorable, and it is straightforward to embed the enrolment protocol in EDHOC with interchanged roles. The details are FFS.


TBD Detail the protocol

TBD CoAP binding. Same as in EDHOC?

TBD name of resource? POST /edhoc?

TBD CoAP Response codes to communicate success or failure of the EALS function?

# Application to 6tisch #


Terminology

The Pledge is EALS client and the Join Registrar/Coordinator is EALS server.


One candidate embedding of EALS into a bootstrapping architecture is as described in {{I-D.ietf-6tisch-minimal-security}}  where the Plegde is EALS/CoAP client, the Join Registrar/Coordinator is EALS/CoAP server and Join Proxy is a CoAP proxy.


Stateless proxy, recap and reference minimal security draft


 BRSKI {{I-D.ietf-anima-bootstrapping-keyinfra}} specifies an automated bootstrapping of a remote secure key infrastructure (BRSKI) using vendor installed X.509 certificate, in combination with a vendor authorized service on the Internet. BRSKI is referencing Enrolment over Secure Transport (EST) {{RFC7030}}. 
 
 
, to enable zero-touch joining of a device in a network domain


One application of EALS is to the BRSKI {{I-D.ietf-anima-bootstrapping-keyinfra}} problem statement. BRSKI deals with automated bootstrapping of new devices using vendor installed X.509 certificate, in combination with a vendor authorized service on the Internet. The following terminology is used

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

   Bootstrapping a new device can occur using a routable address and a
   cloud service, or using only link-local connectivity, or on limited/
   disconnected networks.  Support for lower security models, including
   devices with minimal identity, is described for legacy reasons but
   not encouraged.  Bootstrapping is complete when the cryptographic
   identity of the new key infrastructure is successfully deployed to
   the device but the established secure connection can be used to
   deploy a locally issued certificate to the device as well.


The problem statement from BRSKI is imported into this document. The limitations of applicability to energy constrained devices due to credential size applies also to this document, and further work is needed to specify certificate formats relevant to constrained devices. Having said that, one rationale for this document is a more optimized message exchange, which is favorable in low-power deployments. Related work include {{I-D.richardson-6tisch-dtsecurity-secure-join}} which addresses the low-power problem statement, and {{I-D.ietf-6tisch-minimal-security}} which describes a one-touch procedure using OSCOAP and EDHOC.



# Architectural Overview # {#architecture}

When a pledge first joins a constrained network, it typically does not have IPv6 connectivity to reach the Join Registrar/Coordinator.
For that reason, pledge communicates with the Join Proxy, a one hop neighbor of the pledge.
Join Proxy statelessly relays the exchanges between the pledge and the Join Registrar/Coordinator.

As in the model of {{I-D.ietf-6tisch-minimal-security}}, the Join Proxy plays the role of a CoAP proxy.
Default CoAP proxy, however, keeps state information in order to relay the response back to the originating client, in this case the pledge. 
To mitigate Denial of Service attacks at the Join Proxy, {{I-D.ietf-6tisch-minimal-security}} mandates the use of a new CoAP option, Stateless-Proxy option, that allows the Join Proxy to operate statelessly.
The proxy adds en-route the state information necessary for its operation as the value of the Stateless-Proxy option.
The value of the Stateless-Proxy option is opaque to the JRC/CoAP server. 
The option is echoed back by the JRC/CoAP server, and consumed by the proxy.
{{arch-overview}} illustrates the operation of the Join Proxy.

~~~~~~~~~~~
+--------+     +-------+       +--------+
| pledge |     |  JP   |       |  JRC   |
|        |     |       |       |        |
+--------+     +-------+       +--------+
    |              |                |
    +------------->|                |       
    |   Request    |                |
    |              |                |
    |              +--------------->|
    |              |     Request    |  Stateless-Proxy: opaque value
    |              |                |
    |              |<---------------+
    |              |    Response    |  Stateless-Proxy: opaque value
    |              |                |
    <--------------+                |    
    |   Response   |                |
    |              |                |
~~~~~~~~~~~
{: #arch-overview title="Overview of the bootstrapping architecture."}
{: artwork-align="center"}



# Security Considerations # {#sec-cons}



# Privacy Considerations #

TODO


# IANA Considerations # {#iana}



# Acknowledgments #

The authors wants to thank  

--- back

# Examples {#examples}


--- fluff

