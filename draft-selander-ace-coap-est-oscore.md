---
title: Protecting EST Payloads with OSCORE
abbrev: EST-OSCORE
docname: draft-selander-ace-coap-est-oscore-latest

# date: 2017-03-14

# stand_alone: true

ipr: trust200902
area: Security
wg: ACE Working Group
kw: Internet-Draft
cat: std

coding: utf-8
pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: G. Selander
        name: Göran Selander
        org: Ericsson AB
        email: goran.selander@ericsson.com
      -
        ins: S. Raza
        name: Shahid Raza
        org: RISE
        email: shahid.raza@ri.se
      -
        ins: M. Furuhed
        name: Martin Furuhed
        org: Nexus
        email: martin.furuhed@nexusgroup.com
      -
        ins: M. Vucinic
        name: Malisa Vucinic
        org: INRIA
        email: malisa.vucinic@inria.fr



normative:

  RFC2119:
  RFC7049:
  RFC7252:
  RFC7959:
  RFC8152:
  I-D.ietf-ace-coap-est:
  I-D.ietf-core-object-security:


informative:

  RFC5272:
  RFC6347:
  RFC7228:
  RFC7030:
  I-D.ietf-6tisch-minimal-security:
  I-D.ietf-ace-oscore-profile:
  I-D.selander-ace-cose-ecdhe:
  I-D.ietf-core-oscore-groupcomm:
  I-D.raza-ace-cbor-certificates:


--- abstract


This document specifies public key certificate enrollment procedures protected with application-layer security protocols suitable for Internet of Things (IoT) deployments. The protocols leverage payload formats defined in Enrolment over Secure Transport (EST) and existing IoT standards including the Constrained Application Protocol (CoAP), Concise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format. 

--- middle

# Introduction  {#intro}

One of the challenges with deploying a Public Key Infrastructure (PKI) for the Internet of Things (IoT) is certificate enrolment, because existing enrolment protocols are not optimized for constrained environments {{RFC7228}}.

One optimization of certificate enrollment targeting IoT deployments is specified in EST-CoAPs ({{I-D.ietf-ace-coap-est}}), which defines a version of Enrolment over Secure Transport {{RFC7030}} for transporting EST payloads over CoAP {{RFC7252}} and DTLS {{RFC6347}}, instead of secured HTTP.

This document describes a method for protecting EST payloads over CoAP or HTTP with OSCORE {{I-D.ietf-core-object-security}}. OSCORE specifies an extension to CoAP which protects the application layer message and can be applied independently of how CoAP messages are transported. OSCORE can also be applied to CoAP-mappable HTTP which enables end-to-end security for mixed CoAP and HTTP transfer of application layer data. Hence EST payloads can be protected end-to-end independent of underlying transport and through proxies translating between between CoAP and HTTP.

OSCORE is designed for constrained environments, building on IoT standards such as CoAP, CBOR {{RFC7049}} and COSE {{RFC8152}}, and has in particular gained traction in settings where message sizes and the number of exchanged messages needs to be kept at a minimum, see e.g. {{I-D.ietf-6tisch-minimal-security}}, or for securing multicast CoAP messages {{I-D.ietf-core-oscore-groupcomm}}. Where OSCORE is implemented and used for communication security, the reuse of OSCORE for other purposes, such as enrolment, reduces the implementation footprint.

In order to protect certificate enrolment with OSCORE, the necessary keying material (notably, the OSCORE Master Secret, see {{I-D.ietf-core-object-security}}) needs to be established between CoAP client and server, e.g. using a key exchange protocol; a trusted third party; or pre-established keys. Different options are allowed and with different properties as is indicated in the next section.

Yet other optimizations to certificate based enrolment are possible further improve the performance of certificate enrolment and certificate based authentication, in particular the use of more compact representations of X.509 certificates such as {{I-D.raza-ace-cbor-certificates}}. 



## EST-CoAPs operational differences

This specification builds on EST-CoAPs {{I-D.ietf-ace-coap-est}} but transport layer security provided by DTLS is replaced, or complemented, by protection of the application layer data. This specification deviates from EST-CoAPs in the following respects:

* The DTLS record layer is replaced, or complemented, with OSCORE.
* The DTLS handshake is replaced, or complemented, with an alternative key establishment, for example:
   * A key exchange protocol, such as EDHOC {{I-D.selander-ace-cose-ecdhe}}. The use of a key exchange protocol completes the analogy with EST-CoAPs, and provides perfect forward secrecy (PFS) of the keys used to protect the EST messages.  However, PFS is not necessary for the enrolment procedure and adds significant overhead in terms of message size and round trips.
   * Trusted third party (TTP) based provisioning, such as the OSCORE profile of ACE {{I-D.ietf-ace-oscore-profile}}. This assumes existing security associations between the client and the TTP, and between the server and the TTP, and reduces the message size and round trips compared to a key exchange protocol.
   * Pre-shared keys (PSK). Although one reason for using a PKI is to avoid managing PSK, applying OSCORE directly with PSK specifically during deployment gives a one round-trip enrolment protocol with low message overhead, thereby further reducing the network load and time for commissioning.
* EST payloads protected by OSCORE can be proxied between constrained networks supporting CoAP/CoAPs and non-constrained networks supporting HTTP/HTTPs with a CoAP-HTTP proxy protection without any security processing in the proxy.


## Terminology   {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}. These
words may also appear in this document in lowercase, absent their
normative meanings.

This document uses terminology from {{I-D.ietf-ace-coap-est}} which in turn is based on {{RFC7030}} and, in turn, on {{RFC5272}}. 

# Protocol Design and Layering 

EST-oscore uses CoAP {{RFC7252}} and Block-Wise {{RFC7959}} to transfer EST messages in the same way as {{I-D.ietf-ace-coap-est}}. Figure 1 below shows the layered EST-oscore architecture. 

~~~~~~~~~~~

+------------------------------------------------+
|          EST request/response messages         |
+------------------------------------------------+
|   CoAP with OSCORE   |   HTTP with OSCORE      |
+------------------------------------------------+
|   UDP  |  DTLS/UDP   |        TLS/TCP          |
+------------------------------------------------+

~~~~~~~~~~~
{: #fig-stack title="EST protected with OSCORE."}
{: artwork-align="center"}

EST-oscore follows closely the EST-coaps and EST design. The message types for simple enroll, reenroll, CA certificate retrieval, CSR Attributes request messages and server-side key generation messages apply. Section references in this paragraph refer to EST-coaps ({{I-D.ietf-ace-coap-est}}): The payload format, message bindings, CoAP response codes, message fragmentation based on Blockwise, and the delayed responses specified in Section 5 apply.  
 

# Discovery and URI

The discovery of EST resources defined in Section 5.1 of {{I-D.ietf-ace-coap-est}}, as well as the new Resource Type defined in Section 9.1 of {{I-D.ietf-ace-coap-est}} apply to EST-oscore. Support for OSCORE is indicated by the "osc" attribute defined in Section 9 of {{I-D.ietf-core-object-security}}, for example:

~~~~~~~~~~~

     REQ: GET /.well-known/core?rt=ace.est.sen

     RES: 2.05 Content
   </est>; rt="ace.est";osc

~~~~~~~~~~~
 
The short EST-coaps URI paths defined in Section 5.1 of {{I-D.ietf-ace-coap-est}} also apply.

# OSCORE-Based Security

EST-oscore depends on the application layer security provided by OSCORE for protecting CoAP and CoAP-mappable HTTP independent of transport. The establishment of keys for OSCORE defines many of the properties of the protocol.

If a key exchange protocols is used, fragmentation of the protocol messages needs to be handled. EDHOC {{I-D.selander-ace-cose-ecdhe}} may be carried in CoAP in which case Block fragmentation can be used.


# Proxying

As is noted Section 6 of {{I-D.ietf-ace-coap-est}}, in real-world deployments, the EST server will not always reside within the CoAP boundary.  The EST-server can exist outside the constrained network in a non-constrained network that does not support CoAP but HTTP, thus requiring an intermediary CoAP-to-HTTP proxy.

Since OSCORE is applicable to CoAP-mappable HTTP the EST payloads can be protected end-to-end between EST client and EST server independent of transport protocol or potential transport layer security which may need to be terminated in the proxy, see Figure {{fig-proxy}}. The signed certification request SHOULD be bound to the OSCORE security context using a derived secret analogously to the use of tls-unique as described in Section 3.5 of {{RFC7030}}. The mappings between CoAP and HTTP referred to in Section 6 of {{I-D.ietf-ace-coap-est}} applies and the additional mappings resulting from the use of OSCORE are specified in Section 11 of {{I-D.ietf-core-object-security}}. 

OSCORE provides security end-to-end between EST Server and EST Client. The use of TLS and DTLS is optional.

~~~~~~~~~~~
                                       Constrained-Node Network
  .---------.                       .----------------------------.
  |   CA    |                       |.--------------------------.|
  '---------'                       ||                          ||
       |                            ||                          ||
   .------.  HTTP   .-----------------.   CoAP   .-----------.  ||
   | EST  |<------->|  CoAP-to-HTTP   |<-------->| EST Client|  ||
   |Server|  (TLS)  |      Proxy      |  (DTLS)  '-----------'  ||
   '------'         '-----------------'                         ||
                                    ||                          ||
       <------------------------------------------------>       ||
                        OSCORE      ||                          ||
                                    |'--------------------------'|
                                    '----------------------------'
~~~~~~~~~~~
{: #fig-proxy title="CoAP-to-HTTP proxy at the CoAP boundary."}
{: artwork-align="center"}


# Security Considerations  {#sec-cons}

TBD

# Privacy Considerations 

TBD

# IANA Considerations  {#iana}


# Acknowledgments 



--- back

# Examples {#examples}

TBD

--- fluff

