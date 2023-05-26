---
title: Protecting EST Payloads with OSCORE
abbrev: EST-oscore
docname: draft-ietf-ace-coap-est-oscore-latest

ipr: trust200902
cat: std
submissiontype: IETF
area: Security
workgroup: ACE Working Group
keyword: Internet-Draft
coding: utf-8
pi: # can use array (if all yes) or hash here
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 2

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
        ins: M. Vučinić
        name: Mališa Vučinić
        org: Inria
        email: malisa.vucinic@inria.fr
      -
        ins: T. Claeys
        name: Timothy Claeys
        email: timothy.claeys@gmail.com


normative:

  RFC2119:
  RFC5869:
  RFC6955:
  RFC8949:
  RFC7252:
  RFC7925:
  RFC7959:
  RFC9052:
  RFC9053:
  RFC8613:
  RFC9052:
  RFC9148:
  I-D.ietf-lake-edhoc:

informative:

  RFC2985:
  RFC2986:
  RFC5272:
  RFC5280:
  RFC5914:
  RFC6024:
  RFC6347:
  RFC9147:
  RFC7228:
  RFC7030:
  RFC8392:
  RFC9031:
  RFC9110:
  RFC9112:
  RFC8446:
  I-D.ietf-core-oscore-groupcomm:
  I-D.ietf-core-oscore-edhoc:
  I-D.ietf-cose-x509:
  I-D.ietf-cose-cbor-encoded-cert:
  I-D.tiloca-core-oscore-capable-proxies:


--- abstract


This document specifies public-key certificate enrollment procedures protected with lightweight application-layer security protocols suitable for Internet of Things (IoT) deployments.
The protocols leverage payload formats defined in Enrollment over Secure Transport (EST) and existing IoT standards including the Constrained Application Protocol (CoAP), Concise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format.

--- middle

# Introduction  {#intro}

One of the challenges with deploying a Public Key Infrastructure (PKI) for the Internet of Things (IoT) is certificate enrollment, because existing enrollment protocols are not optimized for constrained environments {{RFC7228}}.

One optimization of certificate enrollment targeting IoT deployments is specified in EST-coaps ({{RFC9148}}), which defines a version of Enrollment over Secure Transport {{RFC7030}} for transporting EST payloads over CoAP {{RFC7252}} and DTLS {{RFC6347}} {{RFC9147}}, instead of HTTP {{RFC9110}} {{RFC9112}} and TLS {{RFC8446}}.

This document describes a method for protecting EST payloads over CoAP or HTTP with OSCORE {{RFC8613}}.
OSCORE specifies an extension to CoAP which protects messages at the application layer and can be applied independently of how CoAP messages are transported. OSCORE can also be applied to CoAP-mappable HTTP which enables end-to-end security for mixed CoAP and HTTP transfer of application layer data.
Hence EST payloads can be protected end-to-end independent of the underlying transport and through proxies translating between between CoAP and HTTP.

OSCORE is designed for constrained environments, building on IoT standards such as CoAP, CBOR {{RFC8949}} and COSE {{RFC9052}} {{RFC9053}}, and has in particular gained traction in settings where message sizes and the number of exchanged messages need to be kept at a minimum, such as 6TiSCH {{RFC9031}}, or for securing CoAP group messages {{I-D.ietf-core-oscore-groupcomm}}.
Where OSCORE is implemented and used for communication security, the reuse of OSCORE for other purposes, such as enrollment, reduces the code footprint.

In order to protect certificate enrollment with OSCORE, the necessary keying material (notably, the OSCORE Master Secret, see {{RFC8613}}) needs to be established between the EST-oscore client and EST-oscore server.
For this purpose we assume by default the use of the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}, although pre-shared OSCORE keying material would also be an option.

Other ways to optimize the performance of certificate enrollment and certificate based authentication described in this draft include the use of:

* Compact representations of X.509 certificates (see {{I-D.ietf-cose-cbor-encoded-cert}})
* Certificates by reference (see {{I-D.ietf-cose-x509}})
* Compact, CBOR representations of EST payloads (see {{I-D.ietf-cose-cbor-encoded-cert}})

## Operational Differences with EST-coaps  {#operational}

The protection of EST payloads defined in this document builds on EST-coaps {{RFC9148}} but transport layer security is replaced, or complemented, by protection of the transfer- and application layer data (i.e., CoAP message fields and payload).
This specification deviates from EST-coaps in the following respects:

* The DTLS record layer is replaced by, or complemented with, OSCORE.
* The DTLS handshake is replaced by, or complemented with, the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}, and makes use of the following features:
   * Authentication based on certificates is complemented with  authentication based on raw public keys.
   * Authentication based on signature keys is complemented with authentication based on static Diffie-Hellman keys, for certificates/raw public keys.
   * Authentication based on certificate by value is complemented with authentication based on certificate/raw public keys by reference.
* The EST payloads protected by OSCORE can be proxied between constrained networks supporting CoAP/CoAPs and non-constrained networks supporting HTTP/HTTPs with a CoAP-HTTP proxy protection without any security processing in the proxy (see {{proxying}}).
The concept "Registrar" and its required trust relation with the EST server as described in Section 5 of {{RFC9148}} is therefore not applicable.

So, while the same authentication scheme (Diffie-Hellman key exchange authenticated with transported certificates) and the same EST payloads as EST-coaps also apply to EST-oscore, the latter specifies other authentication schemes and a new matching EST function.
The reason for these deviations is that a significant overhead can be removed in terms of message sizes and round trips by using a different handshake, public key type or transported credential, and those are independent of the actual enrollment procedure.

# Terminology   {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.
These words may also appear in this document in lowercase, absent their normative meanings.

This document uses terminology from {{RFC9148}} which in turn is based on {{RFC7030}} and, in turn, on {{RFC5272}}.

The term "Trust Anchor" follows the terminology of {{RFC6024}}:
"A trust anchor represents an authoritative entity via a public key and associated data.
The public key is used to verify digital signatures, and the associated data is used to constrain the types of information for which the trust anchor is authoritative."
One example of specifying more compact alternatives to X.509 certificates for exchanging trust anchor information is provided by the TrustAnchorInfo structure of {{RFC5914}}, the mandatory parts of which essentially is the SubjectPublicKeyInfo structure {{RFC5280}}, i.e., an algorithm identifier followed by a public key.

# Authentication

This specification replaces, or complements, the DTLS handshake in EST-coaps with the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}.
During initial enrollment, the EST-oscore client and server run EDHOC {{I-D.ietf-lake-edhoc}} to authenticate and establish the OSCORE Security Context used to protect the messages conveying EST payloads.

The EST-oscore client MUST play the role of the EDHOC Initiator.
The EST-oscore server MUST play the role of the EDHOC Responder.

The EST-oscore clients and servers must perform mutual authentication.
The EST server and EST client are responsible for ensuring that an acceptable cipher suite is negotiated.
The client must authenticate the server before accepting any server response.
The server must authenticate the client.
These requirements are fullfilled when using EDHOC {{I-D.ietf-lake-edhoc}}.

The server must also provide relevant information to the CA for decision about issuing a certificate.

## EDHOC

EDHOC supports authentication with certificates/raw public keys (referred to as "credentials"), and the credentials may either be transported in the protocol, or referenced.
This is determined by the identifier of the credential of the endpoint, ID_CRED_x for x= Initiator/Responder, which is transported in an EDHOC message.
This identifier may be the credential itself (in which case the credential is transported), or a pointer such as a URI to the credential (e.g., x5u, see {{I-D.ietf-cose-x509}}) or some other identifier which enables the receiving endpoint to retrieve the credential.

## Certificate-based Authentication

EST-oscore, like EST-coaps, supports certificate-based authentication between the EST client and server.
In this case the client MUST be configured with an Implicit or Explicit Trust Anchor (TA) {{RFC7030}} database, enabling the client to authenticate the server.
During the initial enrollment the client SHOULD populate its Explicit TA database and use it for subsequent authentications.

The EST client certificate SHOULD conform to {{RFC7925}}.
The EST client and/or EST server certificate MAY be a (natively signed) CBOR certificate {{I-D.ietf-cose-cbor-encoded-cert}}.

## Channel Binding {#channel-binding}

The {{RFC5272}} specification describes proof-of-possession as the ability of a client to prove its possession of a private key which is linked to a certified public key.
In case of signature key, a proof-of-possession is generated by the client when it signs the PKCS#10 Request during the enrollment phase.
Connection-based proof-of-possession is OPTIONAL for EST-oscore clients and servers, and it is supported when EDHOC is executed prior to enrollment.
Connection-based proof-of-possession is not supported when pre-shared OSCORE context is used.

When EDHOC is executed prior to enrollment, the client can use the EDHOC_Exporter API to extract channel-binding information and provide a connection-based proof-of possession.
Channel-binding information is obtained as follows

 edhoc-unique = EDHOC_Exporter(TBD1, "EDHOC Unique", length),

 where TBD1 is a registered label from the EDHOC Exporter Label registry, length equals the desired length of the edhoc-unique byte string.
Unless otherwise indicated by an application profile, the length SHOULD be set to 32 bytes.
The client then adds the edhoc-unique byte string as a challengePassword (see Section 5.4.1 of {{RFC2985}}) in the attributes section of the PKCS#10 Request {{RFC2986}} to prove to the server that the authenticated EDHOC client is in possession of the private key associated with the certification request, and signed the certification request after the EDHOC session was established.

## Optimizations

* The last message of the EDHOC protocol, message_3, MAY be combined with an OSCORE request, enabling authenticated Diffie-Hellman key exchange and a protected CoAP request/response (which may contain an enrolment request and response) in two round trips {{I-D.ietf-core-oscore-edhoc}}.

* The certificates MAY be compressed, e.g., using the CBOR encoding defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

* The client certificate MAY be referenced instead of transported {{I-D.ietf-cose-x509}}.
The EST-oscore server MAY use information in the credential identifier field of the EDHOC message (ID_CRED_x) to access the EST-oscore client certificate, e.g., in a directory or database provided by the issuer.
In this case the certificate may not need to be transported over a constrained link between EST client and server.

* Conversely, the response to the PKCS#10 request MAY specify a reference to the enrolled certificate rather than the certificate itself.
The EST-oscore server MAY in the enrolment response to the EST-oscore client include a pointer to a directory or database where the certificate can be retrieved.

# Protocol Design and Layering
EST-oscore uses CoAP {{RFC7252}} and Block-Wise {{RFC7959}} to transfer EST messages in the same way as {{RFC9148}}.
Instead of DTLS record layer, OSCORE {{RFC8613}} is used to protect the messages conveying the EST payloads.
External Authorization Data (EAD) fields of EDHOC are intentionally not used to carry EST payloads because EDHOC needs not be executed in the case of re-enrollment.
The DTLS handshake is complemented by or replaced with EDHOC {{I-D.ietf-lake-edhoc}}.
{{fig-stack}} below shows the layered EST-oscore architecture.
Note that {{fig-stack}} does not illustrate the potential use of DTLS.

~~~~~~~~~~~

             +----------------+
             |  EST messages  |
+------------+----------------+
|    EDHOC   |    OSCORE      |
+------------+----------------+
|       CoAP or HTTP          |
+-----------------------------+
|        UDP or TCP           |
+-----------------------------+

~~~~~~~~~~~
{: #fig-stack title="EST protected with OSCORE."}
{: artwork-align="center"}

EST-oscore follows much of the EST-coaps and EST design.

## Discovery and URI     {#discovery}
The discovery of EST resources and the definition of the short EST-coaps URI paths specified in Section 4.1 of {{RFC9148}}, as well as the new Resource Type defined in Section 8.2 of {{RFC9148}} apply to EST-oscore.
Support for OSCORE is indicated by the "osc" attribute defined in Section 9 of {{RFC8613}}.

Example:

~~~~~~~~~~~

     REQ: GET /.well-known/core?rt=ace.est.sen

     RES: 2.05 Content
   </est>; rt="ace.est.sen";osc

~~~~~~~~~~~

The use of the "osc" attribute is REQUIRED.
In scenarios where OSCORE and DTLS are combined, the absence of the "osc" attribute might wrongly suggest that the EST server is actually using EST-coaps, because of the scheme "coaps", when it is using EST-oscore.

## Mandatory/optional EST Functions {#est-functions}
The EST-oscore specification has the same set of required-to-implement functions as EST-coaps.
The content of {{table_functions}} is adapted from Section 4.2 in {{RFC9148}} and uses the updated URI paths (see {{discovery}}).

| EST functions  | EST-oscore implementation   |
| /crts          | MUST                        |
| /sen           | MUST                        |
| /sren          | MUST                        |
| /skg           | OPTIONAL                    |
| /skc           | OPTIONAL                    |
| /att           | OPTIONAL                    |
{: #table_functions cols="l l" title="Mandatory and optional EST-oscore functions"}

### /crts {#crts}

EST-coaps provides the /crts operation.
A successful request from the client to this resource will be answered with a bag of certificates which is subsequently installed in the Explicit TA.

A trust anchor is commonly a self-signed certificate of the CA public key.
In order to reduce transport overhead, the trust anchor could be just the CA public key and associated data (see {{terminology}}), e.g., the SubjectPublicKeyInfo, or a public key certificate without the signature.
In either case they can be compactly encoded, e.g. using CBOR encoding {{I-D.ietf-cose-cbor-encoded-cert}}.

## Payload formats
Similar to EST-coaps, EST-oscore allows transport of the ASN.1 structure of a given Media-Type in binary format.
In addition, EST-oscore uses the same CoAP Content-Format identifiers when transferring EST requests and responses.
{{table_mediatypes}} summarizes the information from Section 4.3 in {{RFC9148}}.

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
{: #table_mediatypes cols="l l" title="EST functions and the associated CoAP Content-Format identifiers"}

Content-Format 281 MUST be supported by EST-oscore servers.
Servers MAY also support Content-Format 287.
It is up to the client to support only Content-Format 281, 287 or both.
As indicated in {{Section 4.3 of RFC9148}}, the client will use a CoAP Accept Option in the request to express the preferred response Content-Format.
If an Accept Option is not included in the request, the client is not expressing any preference and the server SHOULD choose format 281.

The generated response for /skg and /skc requests contains two parts: certificate and the corresponding private key.
{{Section 4.8 of RFC9148}} specifies that the private key in response to /skc request may be either an encrypted (PKCS #7) or unencrypted (PKCS #8) key, depending on whether the CSR request included SMIMECapabilities.

Due to the use of OSCORE, which protects the communication between the EST client and the EST server end-to-end, it is possible to return the private key to /skc as an unencrypted PKCS #8 object (Content-Format identifier 284).
Therefore, when making the CSR to /skc, the EST client MUST NOT include SMIMECapabilities.
As a consequence, the response to /skc is an unencrypted PKCS #8 object.

{{table_cft_skg_skc}} summarizes the Content-Format identifiers used in responses to /skg and /skc.

| Function | Response, Part 1 | Response, Part 2 |
| /skg     | 284              | 281              |
| /skc     | 284              | 287              |
{: #table_cft_skg_skc cols="l l" title="Response Content-Format identifiers for /skg and /skc"}

## Message Bindings
Note that the EST-oscore message characteristics are identical to those specified in Section 4.4 of {{RFC9148}}.
It is therefore required that

  * The EST-oscore endpoints support delayed responses
  * The endpoints supports the following CoAP options: OSCORE, Uri-Host, Uri-Path, Uri-Port, Content-Format, Block1, Block2, and Accept.
  * The EST URLs based on https:// are translated to coap://, but with mandatory use of the CoAP OSCORE option.
    In case DTLS is additionally used, the translation target is the scheme "coaps", instead of "coap".

## CoAP response codes
See Section 4.5 in {{RFC9148}}.

## Message fragmentation

The EDHOC key exchange is optimized for message overhead, in particular the use of static DH keys instead of signature keys for authentication (e.g., method 3 of {{I-D.ietf-lake-edhoc}}).
Together with various measures listed in this document such as CBOR-encoded payloads {{RFC8949}}, CBOR certificates {{I-D.ietf-cose-cbor-encoded-cert}}, certificates by reference ({{optimizations}}), and trust anchors without signature ({{crts}}), a significant reduction of message sizes can be achieved.

Nevertheless, depending on the application, the protocol messages may become larger than the available frame size thus resulting in fragmentation and, in resource constrained networks such as IEEE 802.15.4 where throughput is limited, fragment loss can trigger costly retransmissions.

It is RECOMMENDED to prevent IP fragmentation, since it involves an error-prone datagram reassembly.
To limit the size of the CoAP payload, this specification mandates the implementation of the CoAP option Block1 and Block2 to enforce the fragmentation mechanism defined in {{RFC7959}} as described in Section 4.6 of {{RFC9148}}.

## Delayed Responses
See Section 4.7 in {{RFC9148}}.

## Enrollment of Static DH Keys

This section specifies how the EST client enrolls a static DH key.
Because a DH key pair cannot be used for signing operations, the EST client attempting to enroll a DH key must use an alternative proof-of-possesion algorithm.
The EST client obtained the CA certs including the CA's DH certificate using the /crts function.
The certificate indicates the DH group parameters which MUST be respected by the EST client when generating its own DH key pair.
The EST client prepares the PKCS #10 object and computes a MAC by following the steps in Section 4 of {{RFC6955}}.
The Key Derivation Function (KDF) and the MAC MUST be set to the HDKF and HMAC algorithms used by OSCORE.
As per {{RFC8613}}, the HKDF MUST be one of the HMAC-based HKDF {{RFC5869}} algorithms defined for COSE {{RFC9052}}.
The KDF and MAC is thus defined by the hash algorithm used by OSCORE in HKDF and HMAC, which by default is SHA-256.
When EDHOC is used, then the hash algorithm is the application hash algorithm of the selected cipher suite.

# HTTP-CoAP Proxy {#proxying}
As noted in Section 5 of {{RFC9148}}, in real-world deployments, the EST server will not always reside within the CoAP boundary.
The EST-server can exist outside the constrained network in a non-constrained network that supports HTTP but not CoAP, thus requiring an intermediary CoAP-to-HTTP proxy.

Since OSCORE is applicable to CoAP-mappable HTTP (see Section 11 of {{RFC8613}}) the messages conveying the EST payloads can be protected end-to-end between the EST client and EST server, irrespective of transport protocol or potential transport layer security which may need to be terminated in the proxy, see {{fig-proxy}}.
Therefore the concept "Registrar" and its required trust relation with EST server as described in Section 5 of {{RFC9148}} is not applicable.

The mappings between CoAP and HTTP referred to in Section 8.1 of {{RFC9148}} apply, and additional mappings resulting from the use of OSCORE are specified in Section 11 of {{RFC8613}}.

OSCORE provides end-to-end security between EST Server and EST Client.
The additional use of TLS and DTLS is optional.
If a secure association is needed between the EST Client and the CoAP-to-HTTP Proxy, this may also rely on OSCORE {{I-D.tiloca-core-oscore-capable-proxies}}.

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

TBD: Compare with RFC9148

TBD: Channel binding security considerations: 3SHAKE attack and EDHOC.

# Privacy Considerations

TBD

# IANA Considerations  {#iana}

## EDHOC Exporter Label Registry

IANA is requested to register the following entry in the "EDHOC Exporter Label" registry under the group name "Ephemeral Diffie-Hellman Over COSE (EDHOC).

~~~~~~~~~~~

+-------------+------------------------------+-------------------+
| Label       | Description                  | Reference         |
+=============+==============================+===================+
| TBD1        | EDHOC unique                 | [[this document]] |
+-------------+------------------------------+-------------------+

~~~~~~~~~~~
{: #fig-exporter-label title="EDHOC Exporter Label"}

# Acknowledgments

--- fluff
