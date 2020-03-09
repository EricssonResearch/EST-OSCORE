---
title: Protecting EST Payloads with OSCORE
abbrev: EST-oscore
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
      -
        ins: T. Claeys
        name: Timothy Claeys
        org: INRIA
        email: timothy.claeys@inria.fr


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
  RFC8392:
  RFC8613:
  I-D.ietf-6tisch-minimal-security:
  I-D.ietf-ace-oscore-profile:
  I-D.ietf-ace-oauth-authz:
  I-D.selander-ace-cose-ecdhe:
  I-D.ietf-core-oscore-groupcomm:
  I-D.raza-ace-cbor-certificates:


--- abstract


This document specifies public-key certificate enrollment procedures protected with application-layer security protocols suitable for Internet of Things (IoT) deployments. The protocols leverage payload formats defined in Enrollment over Secure Transport (EST) and existing IoT standards including the Constrained Application Protocol (CoAP), Concise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format. 

--- middle

# Introduction  {#intro}

One of the challenges with deploying a Public Key Infrastructure (PKI) for the Internet of Things (IoT) is certificate enrollment, because existing enrollment protocols are not optimized for constrained environments {{RFC7228}}.

One optimization of certificate enrollment targeting IoT deployments is specified in EST-coaps ({{I-D.ietf-ace-coap-est}}), which defines a version of Enrollment over Secure Transport {{RFC7030}} for transporting EST payloads over CoAP {{RFC7252}} and DTLS {{RFC6347}}, instead of secured HTTP.

This document describes a method for protecting EST payloads over CoAP or HTTP with OSCORE {{I-D.ietf-core-object-security}}. OSCORE specifies an extension to CoAP which protects the application layer message and can be applied independently of how CoAP messages are transported. OSCORE can also be applied to CoAP-mappable HTTP which enables end-to-end security for mixed CoAP and HTTP transfer of application layer data. Hence EST payloads can be protected end-to-end independent of underlying transport and through proxies translating between between CoAP and HTTP.

OSCORE is designed for constrained environments, building on IoT standards such as CoAP, CBOR {{RFC7049}} and COSE {{RFC8152}}, and has in particular gained traction in settings where message sizes and the number of exchanged messages needs to be kept at a minimum, see e.g. {{I-D.ietf-6tisch-minimal-security}}, or for securing multicast CoAP messages {{I-D.ietf-core-oscore-groupcomm}}. Where OSCORE is implemented and used for communication security, the reuse of OSCORE for other purposes, such as enrollment, reduces the implementation footprint.

In order to protect certificate enrollment with OSCORE, the necessary keying material (notably, the OSCORE Master Secret, see {{I-D.ietf-core-object-security}}) needs to be established between CoAP client and server, e.g. using a key exchange protocol; a trusted third party; or pre-established keys. Different options are allowed and with different properties as is indicated in the next section.

Yet other optimizations to certificate based enrollment are possible further improve the performance of certificate enrollment and certificate based authentication, in particular the use of more compact representations of X.509 certificates such as {{I-D.raza-ace-cbor-certificates}}. 


## EST-coaps operational differences {#operational}

This specification builds on EST-coaps {{I-D.ietf-ace-coap-est}} but transport layer security provided by DTLS is replaced, or complemented, by protection of the application layer data. This specification deviates from EST-coaps in the following respects:

* The DTLS record layer is replaced, or complemented, with OSCORE.
* The DTLS handshake is replaced, or complemented, with the EDHOC key exchange protocol {{I-D.selander-ace-cose-ecdhe}}. The use of certificate authentication with EDHOC completes the analogy with EST-coaps, and provides perfect forward secrecy (PFS) of the keys used to protect the EST messages. However, PFS is not necessary for the enrollment procedure and adds significant overhead in terms of message size and round trips. EDHOC can also leverage its support for static Diffie-Hellman keys. The latter implies that the certificates contain public keys that can be used for a Diffie-Hellman key exchange.
   * {{alternative-auth}} discusses alternative authenticated key exchange methods.
* The EST payloads protected by OSCORE can be proxied between constrained networks supporting CoAP/CoAPs and non-constrained networks supporting HTTP/HTTPs with a CoAP-HTTP proxy protection without any security processing in the proxy.


# Terminology   {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}. These
words may also appear in this document in lowercase, absent their
normative meanings.

This document uses terminology from {{I-D.ietf-ace-coap-est}} which in turn is based on {{RFC7030}} and, in turn, on {{RFC5272}}. 

# OSCORE and Authenticated Key Establishment
EST-oscore clients and servers MUST perform mutual authentication before EST-oscore functions and services can be accessed. Prior to the initial enrollment the client MUST be configured with an Implicit Trust Anchor (TA) {{RFC7030}} database, enabling the client to authenticate the server. During the initial enrollment the client SHOULD populate its Explicit TA database and use it for subsequent authentications. 

The EST-coaps specification {{I-D.ietf-ace-coap-est}} only supports certificate-based authentication during the DTLS handshake between the EST-coaps server and the client. This specification replaces the DTLS handshake with the certificate-based EDHOC key exchange protocol but provides additional authenticated methods in the {{alternative-auth}} 

The {{RFC5272}} specification describes proof-of-identity as the ability of an endpoint, i.e., client, to prove its possession of a private key which is linked to a certified public key. Additionally, the RFC details how to use channel-binding information (extracted from the underlying TLS layer) to link the proof-of-identity to a proof-of-possession. A proof-of-possession is generated by the client when it signs the PKCS#10 Request during the enrollment phase. Connection-based proof-of-possession is OPTIONAL for EST-oscore clients and servers. In case of certificate-based EDHOC key establishment, a set of actions are required which are further described below.

## EDHOC   {#edhoc}
The EST-oscore client and server use the EDHOC key establishment protocol to populate the OSCORE security context. The endpoints MUST use public key certificates to perform mutual authentication. During the initial enrollment the Implicit TA database MUST contain certificates capable of authenticating the EST-oscore server. When the EST-oscore client issues a request to the /crts endpoint of the EST server, it SHALL return a bag of certificates to be installed in the Explicit TA database. 

The cryptographic material used for EST-oscore client authentication can either be

 * previously issued certificates (e.g., an existing certificate issued by the EST server); this could be a common case for simple re-enrollment of clients.
 * previously installed certificates (e.g., installed by the manufacturer). Manufacturer installed certificates are expected to have a very long life, as long as the device, but under some conditions could expire. In that case, the server MAY authenticate a client certificate against its trust store although the certificate is expired ({{sec-cons}}).
 
 When desired the client can use the EDHOC-Exporter API to extract channel-binding information and provide a connection-based proof-of possession. Channel-binding information is obtained as follows 
 
 edhoc-unique = EDHOC-Exporter("EDHOC Unique", length),
 
 where length equals the desired length of the edhoc-unique byte string. The client then adds the edhoc-unique byte string as a ChallengePassword in the attributes section of the PKCS#10 Request to prove that the client is indeed in control of the private key at the time of the EDHOC key exchange.


# Protocol Design and Layering 
EST-oscore uses CoAP {{RFC7252}} and Block-Wise {{RFC7959}} to transfer EST messages in the same way as {{I-D.ietf-ace-coap-est}}. {{fig-stack}} below shows the layered EST-oscore architecture. 

~~~~~~~~~~~

+------------------------------------------------+
|          EST request/response messages         |
+------------------------------------------------+
|   CoAP with OSCORE   |   HTTP with OSCORE      |
+------------------------------------------------+
|   UDP  |  DTLS/UDP   |   TCP   |   TLS/TCP     |
+------------------------------------------------+

~~~~~~~~~~~
{: #fig-stack title="EST protected with OSCORE."}
{: artwork-align="center"}

EST-oscore follows closely the EST-coaps and EST design. 
 

## Discovery and URI     {#discovery}
The discovery of EST resources and the definition of the short EST-coaps URI paths specified in Section 5.1 of {{I-D.ietf-ace-coap-est}}, as well as the new Resource Type defined in Section 9.1 of {{I-D.ietf-ace-coap-est}} apply to EST-oscore. Support for OSCORE is indicated by the "osc" attribute defined in Section 9 of {{I-D.ietf-core-object-security}}, for example:

~~~~~~~~~~~

     REQ: GET /.well-known/core?rt=ace.est.sen

     RES: 2.05 Content
   </est>; rt="ace.est";osc

~~~~~~~~~~~
 
## Mandatory/optional EST Functions
The EST-oscore specification has the same set of required-to-implement functions as EST-coaps. The content of {{table_functions}} is adapted from Section 5.2 in {{I-D.ietf-ace-coap-est}} and uses the updated URI paths (see {{discovery}}).

| EST functions  | EST-oscore implementation   |
| /crts          | MUST                        |
| /sen           | MUST                        |
| /sren          | MUST                        |
| /skg           | OPTIONAL                    | 
| /skc           | OPTIONAL                    |
| /att           | OPTIONAL                    |
{: #table_functions cols="l l" title="Mandatory and optional EST-oscore functions"}

## Payload formats
Similar to EST-coaps, EST-oscore transports the ASN.1 structure of a given Media-Type in binary format. In addition, it uses the same CoAP Content-Format Options to transport EST requests and responses. {{table_mediatypes}} summarizes the information from Section 5.3 in {{I-D.ietf-ace-coap-est}}.

|  URI  | Content-Format                                       | #IANA |
| /crts | N/A                                            (req) |   -   |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /sen  | application/pkcs10                             (req) |  286  |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /sren | application/pkcs10                             (req) |  286  |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /skg  | application/pkcs10                             (req) |  286  |
|       | application/multipart-core                     (res) |   62  |
| /skc  | application/pkcs10                             (req) |  286  |
|       | application/multipart-core                     (res) |   62  |
| /att  | N/A                                            (req) |   -   |
|       | application/csrattrs                           (res) |  285  |
{: #table_mediatypes cols="l l" title="EST functions and there associated Media-Type and IANA numbers"}

NOTE: CBOR is becoming a de facto encoding scheme in IoT settings. There is already work in progress on CBOR encoding of X.509 certificates {{I-D.raza-ace-cbor-certificates}}, and this can be extended to other EST messages. 


## Message Bindings
The EST-oscore message characteristics are identical to those specified in Section 5.4 of {{I-D.ietf-ace-coap-est}}. It is RECOMMENDED that

  * The EST-oscore endpoints support delayed responses
  * The endpoints supports the following CoAP options: OSCORE, Uri-Host, Uri-Path, Uri-Port, Content-Format, Block1, Block2, and Accept.
  * The EST URLs based on https:// are translated to coap://, but with mandatory use of the CoAP OSCORE option.


## CoAP response codes
See Section 5.5 in {{I-D.ietf-ace-coap-est}}.

## Message fragmentation
The EDHOC key exchange with asymmetric keys and certificates for authentication can result in large messages. It is RECOMMENDED to prevent IP fragmentation, since it involves an error-prone datagram reconstitution. In addition, this specification targets resource constrained networks such as IEEE 802.15.4 where throughput is limited and fragment loss can trigger costly retransmissions. Even though ECC-based certificates are an improvement over the RSA or DSA counterparts, they can still amount to roughly a 1000 bytes per certificate depending on the used algorithms, curves, OIDs, Subject Alternative Names (SAN) and cert fields. Additionally, in response to a client request to /crts an EST-oscore server might answer with multiple certificates. This specification employs the CoAP Block1 and Block2 fragmentation mechanisms as described in Section 5.6 of {{I-D.ietf-ace-coap-est}} to limit the size of the CoAP payload.


## Delayed Responses
See Section 5.7 in {{I-D.ietf-ace-coap-est}}.

# HTTP-CoAP registrar 
As is noted in Section 6 of {{I-D.ietf-ace-coap-est}}, in real-world deployments, the EST server will not always reside within the CoAP boundary.  The EST-server can exist outside the constrained network in a non-constrained network that does not support CoAP but HTTP, thus requiring an intermediary CoAP-to-HTTP proxy.

Since OSCORE is applicable to CoAP-mappable HTTP the EST payloads can be protected end-to-end between EST client and EST server independent of transport protocol or potential transport layer security which may need to be terminated in the proxy, see {{fig-proxy}}. When using the EDHOC key exchange protocol to establish a shared OSCORE security context, PKCS#10 request MAY be bound to the OSCORE security context using the procedure described in {{edhoc}}. The mappings between CoAP and HTTP referred to in Section 6 of {{I-D.ietf-ace-coap-est}} apply and the additional mappings resulting from the use of OSCORE are specified in Section 11 of {{I-D.ietf-core-object-security}}. 

OSCORE provides end-to-end security between EST Server and EST Client. The use of TLS and DTLS is optional.

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

# Other Authentication Methods {#alternative-auth}

## TTP Assisted Authentication
Trusted third party (TTP) based provisioning, such as the OSCORE profile of ACE {{I-D.ietf-ace-oscore-profile}} assumes existing security associations between the client and the TTP, and between the server and the TTP. This setup allows for reduced message overhead and round trips compared to the full-fledged EDHOC key exchange. Following the ACE terminology the TTP plays the role of the Authorization Server (AS), the EST-oscore client corresponds to the ACE client and the EST-oscore server is the ACE Resource Server (RS).

~~~~~~~~~~~

+------------+                               +------------+
|            |                               |            |
|            | ---(A)- Token Request ------> |  Trusted   |
|            |                               |   Third    |
|            | <--(B)- Access Token -------  | Party (AS) |
|            |                               |            |
|            |                               +------------+
| EST-oscore |                                  |     ^
|   Client   |                                 (F)   (E)
|(ACE Client)|                                  V     |
|            |                               +------------+
|            |                               |            | 
|            | -(C)- Token + EST Request --> | EST-oscore | 
|            |                               | server (RS)|
|            | <--(D)--- EST response ------ |            |
|            |                               |            | 
+------------+                               +------------+


~~~~~~~~~~~
{: #fig-ttp title="Accessing EST services using a TTP for authenticated key establishment and authorization."}
{: artwork-align="center"}

During initial enrollment the EST-oscore client uses its existing security association with the TTP, which replaces the Implicit TA database, to establish an authenticated secure channel. The {{I-D.ietf-ace-oscore-profile}} ACE profile RECOMMENDS the use of OSCORE between client and TTP (AS), but TLS or DTLS MAY be used additionally or instead. The client requests an access token at the TTP corresponding the EST service it wants to access. If the client request was invalid, or not authorized according to the local EST policy, the AS returns an error response as described in Section 5.6.3 of {{I-D.ietf-ace-oauth-authz}}. In its responses the TTP (AS) SHOULD signal that the use of OSCORE is REQUIRED for a specific access token as indicated in section 4.3 of {{I-D.ietf-ace-oscore-profile}}. This means that the EST-oscore client MUST use OSCORE towards all EST-oscore servers (RS) for which this access token is valid, and follow Section 4.3 in {{I-D.ietf-ace-oscore-profile}} to derive the security context to run OSCORE. The ACE OSCORE profile RECOMMENDS the use of CBOR web token (CWT) as specified in {{RFC8392}}. The TTP (AS) MUST also provision an OSCORE security context to the EST-oscore client and EST-oscore server (RS), which is then used to secure the subsequent messages between the client and the server.  The details on how to transfer the OSCORE contexts are described in section 3.2 of {{I-D.ietf-ace-oscore-profile}}.

Once the client has retrieved the access token it follows the steps in {{I-D.ietf-ace-oscore-profile}} to install the OSCORE security context and presents the token to the EST-oscore server. The EST-oscore server installs the corresponding OSCORE context and can either verify the validity of the token locally or request a token introspection at the TTP. In either case EST policy decisions, e.g., which client can request enrollment or reenrollment, can be implemented at the TTP. Finally the EST-oscore client receives a response from the EST-oscore server.


## PSK Based Authentication
Another method to bootstrap EST services requires a pre-shared OSCORE context between the EST-oscore client and EST-oscore server. Authentication using the Implicit TA is no longer required since the shared security context authenticates both parties. The client and server have access to the same master secret as well as

  * a server identifier
  * a client identifier
  * a context identifier
  * an AEAD algorithm
  * an HKDF algorithm
  * a salt
  * a replay window type and size

The OSCORE specification {{RFC8613}} makes use of the 'kid context' header parameter in the COSE object to indicate which OSCORE security context to use.


# CBOR Encoding of EST Payloads



--- fluff

