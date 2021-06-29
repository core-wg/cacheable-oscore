---
title: "Cacheable OSCORE"
docname: draft-amsuess-core-cachable-oscore-latest
ipr: trust200902
stand_alone: true
cat: std
wg: CoRE Working Group
kw: CoAP, OSCORE, multicast, caching, proxy
author:
- ins: C. Amsüss
  name: Christian Amsüss
  country: Austria
  email: christian@amsuess.com
-
  ins: M. Tiloca
  name: Marco Tiloca
  org: RISE AB
  street: Isafjordsgatan 22
  city: Kista
  code: SE-16440 Stockholm
  country: Sweden
  email: marco.tiloca@ri.se

normative:
  I-D.ietf-core-groupcomm-bis:
  I-D.ietf-core-oscore-groupcomm:
  I-D.ietf-cose-rfc8152bis-struct:
  I-D.ietf-cose-rfc8152bis-algs:
  RFC2119:
  RFC7252:
  RFC8132:
  RFC8174:
  RFC8613:
  COSE.Algorithms:
    author:
      org: IANA
    date: false
    title: COSE Algorithms
    target: https://www.iana.org/assignments/cose/cose.xhtml#algorithms

informative:
  RFC7641:
  I-D.ietf-core-echo-request-tag:
  I-D.ietf-ace-key-groupcomm-oscore:
  I-D.amsuess-lwig-oscore:
  I-D.ietf-core-observe-multicast-notifications:

--- abstract

Group communication with the Constrained Application Protocol (CoAP) can be secured end-to-end using Group Object Security for Constrained RESTful Environments (Group OSCORE), also across untrusted intermediary proxies. However, this sidesteps the proxies' abilities to cache responses from the origin server(s). This specification restores cacheability of protected responses at proxies, by introducing consensus requests which any client in a group can send to one server or multiple servers in the same group.

--- middle

# Introduction {#introduction}

The Constrained Application Protocol (CoAP) {{RFC7252}} supports also group communication, for instance over UDP and IP multicast {{I-D.ietf-core-groupcomm-bis}}. In a group communication environment, exchanged messages can be secured end-to-end by using Group Object Security for Constrained RESTful Environments (Group OSCORE) {{I-D.ietf-core-oscore-groupcomm}}.

Requests and responses protected with the group mode of Group OSCORE can be read by all group members, i.e. not only by the intended recipient(s), thus achieving group-level confidentiality.

This allows a trusted intermediary proxy which is also a member of the OSCORE group to populate its cache with responses from origin servers. Later on, the proxy can possibly reply to a request in the group with a response from its cache, if recognized as an eligible server by the client.

However, an untrusted proxy which is not member of the OSCORE group only sees protected responses as opaque, uncacheable ciphertext. In particular, different clients in the group that originate a same plain CoAP request would send different protected requests, as a result of their Group OSCORE processing. Such protected requests cannot yield a cache hit at the proxy, which makes the whole caching of protected responses pointless.

This document addresses this complication and enables cacheability of protected responses, also for proxies that are not members of the OSCORE group and are unaware of OSCORE in general. To this end, it builds on the concept of "consensus request" initially considered in {{I-D.ietf-core-observe-multicast-notifications}}, and defines "deterministic request" as a convenient incarnation of such concept.

Intuitively, given a GET or FETCH plain CoAP request, all clients wishing to send that request are able to deterministically compute the same protected request, using a variation on the pairwise mode of Group OSCORE. It follows that cache hits become possible at the proxy, which can thus serve clients in the group from its cache. Like in {{I-D.ietf-core-observe-multicast-notifications}}, this requires that clients and servers are already members of a suitable OSCORE group.

Cacheability of protected responses is useful also in applications where several clients wish to retrieve the same object.
Some security properties of OSCORE are dispensed with to gain other desirable properties.

## Use cases

When firmware updates are delivered using CoAP,
many similar devices fetch large representations at the same time.
Collecting them at a proxy not only keeps the traffic low,
but also lets the clients ride single file to hide their numbers[SW:EPIV].

\[TODO: include the actual reference "SW:EPIV"\]

When relying on intermediaries to fan out the delivery of multicast data protected end-to-end as in {{I-D.ietf-core-observe-multicast-notifications}}, deterministic requests allow for a more efficient setup, by reducing the amount of message exchanges and enabling early population of cache entries (see {{det-requests-for-notif}}).

## Terminology ## {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Readers are expected to be familiar with terms and concepts of CoAP {{RFC7252}} and its method FETCH {{RFC8132}}, group communication for CoAP {{I-D.ietf-core-groupcomm-bis}}, COSE {{I-D.ietf-cose-rfc8152bis-struct}}{{I-D.ietf-cose-rfc8152bis-algs}}, OSCORE {{RFC8613}}, and Group OSCORE {{I-D.ietf-core-oscore-groupcomm}}.

This document introduces the following new terms.

* Consensus Request: a CoAP request protected with Group OSCORE that can be used repeatedly to access a particular resource, hosted at one or more servers in the OSCORE group.

   A Consensus Request has all the properties relevant to caching, but its transport dependent properties (e.g. Token or Message ID) are not defined. Thus, different requests on the wire can both be said to "be the same Consensus Request" even if they have different Tokens or source addresses.

   The Consensus Request is the reference for request-response binding. Hence, if it does not generate a Consensus Request by itself, the client has still to be able to read and verify any obtained Consensus Request, before using it to verify a bound response.

* Deterministic Client: a fictitious member of an OSCORE group, having no Sender Sequence Number, no asymmetric key pair, and no Recipient Context.

   The Group Manager sets up the Deterministic Client, and assigns it a unique Sender ID as for other group members. Furthermore, the Deterministic Client has only the minimum common set of privileges shared by all group members.

* Deterministic Request: a Consensus Request generated by the Deterministic Client. The use of Deterministic Requests is defined in {{sec-deterministic-requests}}.

* Ticket Request: a Consensus Request generated by the server itself.

  This term is not used in the main document, but is useful in comparison with other applications of consensus requests
  that are generated in a different way than as a Deterministic Request.
  The prototypical Ticket Request is the Phantom Request defined in {{I-D.ietf-core-observe-multicast-notifications}}.

  In {{sec-ticket-requests}}, the term is used to bridge the gap to that draft.

# Deterministic Requests # {#sec-deterministic-requests}

This section defines a method for clients starting from a same plain CoAP request to independently arrive at a same Deterministic Request protected with Group OSCORE.

While the first client sending the Deterministic Request actually reaches the origin server, the response can be cached by an intermediary proxy. Later on, a different client with the same plain CoAP request would send the same Deterministic Request, which will be served from the proxy's cache.

Clients build the unprotected Deterministic Request in a way which is as much reproducible as possible.
This document does not set out full guidelines for minimizing the variation,
but considered starting points are:

* Set the inner Observe option to 0 if the requested resource is described as observable,
  even if no observation is intended (and no outer Observe is set).
  Thus, both observing and non-observing requests can be aggregated into a single request,
  that is upstreamed as an observation at the latest when any observing request reaches the proxy.

<!--
MT: Doesn't this prevent the request from C2 including an outer Observe option to reach the server, and hence for the proxy to start the observation at the server for obtaining future notifications?
-->

* Avoid setting the ETag option in requests on a whim.
  Only set it when there was a recent response with that ETag.
  When obtaining later blocks, do not send the known-stale ETag.

<!--
MT: It refers to "blocks". Does it simpy mean responses as different representation versions? It shouldn't necessarily refer to blocks as intended for the block-wise transfer (which is covered in the following bullet point).
-->
  
* In block-wise transfer, maximally sized large inner blocks (szx=6) should be selected.
  This serves not only to align the clients on consistent cache entries,
  but also helps amortize the additional data transferred in the per-message signatures.

<!--
MT: proposed  s/should be selected/SHOULD be selected
-->
  
  Outer block-wise transfer can then be used if these messages excede a hop's efficiently usable MTU size.

  (If BERT {{?RFC8323}} is usable with OSCORE, its use is fine as well;
  in that case, the server picks a consistent block size for all clients anyway).

<!--
MT: proposed  s/If BERT ... is usable/When BERT ... is used
-->
  
* If padding (see {{sec-padding}}) is used to limit an adversary's ability to deduce requests' content from their length, the requests are padded to reach a total length that should be agreed on among all users of a security context.

<!--
MT: proposed  s/should be agreed/SHOULD be agreed
-->

These only serve to ensure that cache entries are utilized; failure to follow them has no more severe consequences than decreasing the utility and effectiveness of a cache.

## Design Considerations ## {#ssec-deterministic-requests-design}

The hard part is arriving at a consensus pair (key, nonce) to be used with the AEAD cipher for encrypting the Deterministic Request, while also avoiding reuse of the same (key, nonce) pair across different requests.

Diversity can conceptually be enforced by applying a cryptographic hash function to the complete input of the encryption operation over the plain CoAP request (i.e., the AAD and the plaintext of the COSE object), and then using the result as source of uniqueness.
Any non-malleable cryptographically secure hash of sufficient length to make collisions sufficiently unlikely is suitable for this purpose.

A tempting possibility is to use a fixed key, and use the hash as a deterministic AEAD nonce for each Deterministic Request throught the Partial IV component (see {{Section 5.2 of RFC8613}}). However, the 40 bit available for the Partial IV are by far insufficient to ensure that the deterministic nonce is not reused across different Deterministic Requests. Even if the full deterministic AEAD nonce could be set, the sizes used by common algorithms would still be too small.

As a consequence, the proposed method takes the opposite approach, by considering a fixed deterministic AEAD nonce, while generating a different deterministic encryption key for each Deterministic Request. That is, the hash computed over the plain CoAP request is taken as input to the key generation. As an advantage, this approach does not require to transport the computed hash in the OSCORE option.

\[ Note: This has a further positive side effect arising with version -11 of Group OSCORE. That is, since the full encoded OSCORE option is part of the AAD, it avoids a circular dependency from feeding the AAD into the hash computation, which in turn needs crude workarounds like building the full AAD twice, or zeroing out the hash-to-be. \]

## Request-Hash ## {#ssec-request-hash-option}

In order to transport the hash of the plain CoAP request, a new CoAP option is defined, which MUST be supported by clients and servers that support Deterministic Requests.

The option is called Request-Hash. As summarized in {{request-hash-table}}, the Request-Hash option is elective, safe to forward, part of the cache key and repeatable.

~~~
+------+---+---+---+---+--------------+--------+--------+---------+
| No.  | C | U | N | R |     Name     | Format | Length | Default |
+------+---+---+---+---+--------------+--------+--------+---------+
| TBD1 |   |   |   | x | Request-Hash | opaque |  any   | (none)  |
+------+---+---+---+---+--------------+--------+--------+---------+
~~~
{: #request-hash-table title="Request-Hash Option" artwork-align="center"}

The Request-Hash option is identical in all its properties to the Request-Tag option defined in {{I-D.ietf-core-echo-request-tag}}, with the following exceptions:

* It may be arbitrarily long.

  Implementations can limit its length to that of the longest output of the supported hash functions.

* A proxy MAY use any fresh cached response from the selected server to respond to a request with the same Request-Hash (or possibly even if the new request's Request-Hash is a prefix of the cached one).

  This is a potential future optimization which is not mentioned anywhere else yet, and allows clients to elide all other options and payload if it has reason to believe that it can produce a cache hit with the abbreviated request alone.

* It may be present in responses (TBD: Does this affect any other properties?).

* When used with a Deterministic Request, this option is created at message protection time by the sender, and used before message unprotection by the recipient. Therefore, in this use case, it is treated as Class U for OSCORE {{RFC8613}} in requests. In the same application, for responses, it is treated as Class I, and often elided from sending (but reconstructed at the receiver). Other uses of this option can put it into different classes for the OSCORE processing.

## Use of Deterministic Requests {#ssec-use-deterministic-requests}

This section defines how a Deterministic Request is built on the client side and then processed on the server side.

### Pre-Conditions {#sssec-use-deterministic-requests-pre-conditions}

The use of Deterministic Requests in an OSCORE group requires that the interested group members are aware of the Deterministic Client in the group. In particular, they need to know:

* The Sender ID of the Deterministic Client, to be used as 'kid' parameter for the Deterministic Requests. This allows all group members to compute the Sender Key of the Deterministic Client.

* The hash algorithm to use for computing the hash of a plain CoAP request, when producing the associated Deterministic Request.

* Optionally, a creation timestamp associated to the Deterministic Client. This is aligned with the Group Manager that might replace the current Deterministic Client with a new one with a different Sender ID, e.g. to enforce freshness indications without rekeying the whole group.

<!--
MT: Why a creation timestamp and not an expiration timestamp or residual lifetime?
-->

Group members have to obtain this information from the Group Manager. A group member can do that, for instance, when obtaining the group key material upon joining the OSCORE group, or later on as an active member by sending a request to a dedicated resource at the Group Manager. In either case, information on the latest Deterministic Client is returned.

The Group Manager defined in {{I-D.ietf-ace-key-groupcomm-oscore}} can be easily extended to support the provisioning of information about the Deterministic Client;
no such extension has been drafted as of the publication of this draft.

<!--
MT: Should we actually define in this document how the ACE Group Manager is extended to provide such information on the current Deterministic Client?
-->

### Client Processing of Deterministic Request {#sssec-use-deterministic-requests-client-req}

In order to build a Deterministic Request, the client protects the plain CoAP request using the pairwise mode of Group OSCORE (see {{Section 9 of I-D.ietf-core-oscore-groupcomm}}), with the following alterations.

1. When preparing the OSCORE option, the external_aad and the AEAD nonce:

   * The used Sender ID is the Deterministic Client's Sender ID.

   * The used Partial IV is 0.
   
   When preparing the external_aad, the element 'sender_public_key' in the aad_array takes the CBOR simple value Null.

2. The client uses the hash function indicated for the Deterministic Client, and computes a hash H over the following input: the Sender Key of the Deterministic Client, concatenated with the external_aad from step 1, concatenated with the COSE plaintext.

   Note that the payload of the plain CoAP request (if any) is not self-delimiting, and thus hash functions are limited to non-malleable ones.

3. The client derives the deterministic Pairwise Sender Key K as defined in {{Section 2.3.1 of I-D.ietf-core-oscore-groupcomm}}, with the following differences:

   * The Sender Key of the Deterministic Client is used as first argument of the HKDF.

   * The hash H from step 2 is used as second argument of the HKDF, i.e. as a pseudo IKM-Sender computable by all the group members.
   
      Note that an actual IKM-Sender cannot be obtained, since there is no public key associated with the deterministic client, to be used as Sender Public Key and for computing an actual Diffie-Hellman Shared Secret.
      
   * The Sender ID of the Deterministic Client is used as value for the 'id' element of the 'info' parameter used as third argument of the HKDF.

4. The client includes a Request-Hash option in the request to protect, with value set to the hash H from Step 2.

5. The client protects the request using the pairwise mode of Group OSCORE as defined in {{Section 9.3 of I-D.ietf-core-oscore-groupcomm}}, using the AEAD nonce from step 1, the deterministic Pairwise Sender Key K from step 3 as AEAD encryption key, and the finalized AAD.

6. The client sets FETCH as the outer code of the protected request to make it usable for a proxy's cache, even if no observation is requested {{RFC7641}}.

The result is the Deterministic Request to be sent.

Since the encryption key K is derived using material from the whole plain CoAP request, this (key, nonce) pair is only used for this very message, which is deterministically encrypted unless there is a hash collision between two Deterministic Requests.

The deterministic encryption requires the used AEAD algorithm to be deterministic in itself. This is the case for all the AEAD algorithms currently registered with COSE in {{COSE.Algorithms}}. For future algorithms, a flag in the COSE registry is to be added.

Note that, while the process defined above is based on the pairwise mode of Group OSCORE, no information about the server takes part to the key derivation or is included in the AAD. This is intentional, since it allows for sending a deterministic request to multiple servers at once (see {{det-req-one-to-many}}). On the other hand, it requires later checks at the client when verifying a response to a Deterministic Request (see {{ssec-use-deterministic-requests-response}}).

### Server Processing of Deterministic Request {#sssec-use-deterministic-requests-server-req}

Upon receiving a Deterministic Request, a server performs the following actions.

A server that does not support Deterministic Requests would not be able to create the necessary Recipient Context, and thus will fail decrypting the request.

1. If not already available, the server retrieves the information about the Deterministic Client from the Group Manager, and derives the Sender Key of the Deterministic Client.

2. The server actually recognizes the request to be a Deterministic Request, due to the presence of the Request-Hash option and to the 'kid' parameter of the OSCORE option set to the Sender ID of the Deterministic Client.

   If the 'kid' parameter of the OSCORE option specifies a different Sender ID than the one of the Deterministic Client, the server MUST NOT take the following steps, and instead processes the request as per {{Section 9.4 of I-D.ietf-core-oscore-groupcomm}}.

3. The server retrieves the hash H from the Request-Hash option.

4. The server derives a Recipient Context for processing the Deterministic Request. In particular:

   - The Recipient ID is the Sender ID of the Deterministic Client.

   - The Recipient Key is derived as the key K in step 3 of {{sssec-use-deterministic-requests-client-req}}, with the hash H retrieved at the previous step.

5. The server verifies the request using the pairwise mode of Group OSCORE, as defined in {{Section 9.4 of I-D.ietf-core-oscore-groupcomm}}, using the Recipient Context from step 4, with the following differences.

   - The server does not perform replay checks against a Replay Window (see below).

In case of successful verification, the server MUST also perform the following actions, before possibly delivering the request to the application.

* Starting from the recovered plain CoAP request, the server MUST recompute the same hash that the client computed at step 2 of {{sssec-use-deterministic-requests-client-req}}.

   If the recomputed hash value differs from the value retrieved from the Request-Hash option at step 3, the server MUST treat the request as invalid and MAY reply with an unprotected 4.00 (Bad Request) error response. The server MAY set an Outer Max-Age option with value zero. The diagnostic payload MAY contain the string "Decryption failed".

   This prevents an attacker that guessed a valid authentication tag for a given Request-Hash value to poison caches with incorrect responses.

* The server MUST verify that the unprotected request is safe to be processed in the REST sense, i.e. that it has no side effects. If verification fails, the server MUST discard the message and SHOULD reply with a protected 4.01 (Unauthorized) error response.

  Note that some CoAP implementations may not be able to prevent that an application produces side effects from a safe request. This may incur checking whether the particular resource handler is explicitly marked as eligible for processing deterministic requests. An implementation may also have a configured list of requests that are known to be side effect free, or even a pre-built list of valid hashes for all sensible requests for them, and reject any other request.

  These checks replace the otherwise present requirement that the server needs to check the Replay Window of the Recipient Context (see step 5 above), which is inapplicable with the Recipient Context derived at step 4 from the value of the Request-Hash option. The reasoning is analogous to the one in {{I-D.amsuess-lwig-oscore}} to treat the potential replay as answerable, if the handled request is side effect free.

### Response to a Deterministic Request {#ssec-use-deterministic-requests-response}

When treating a response to a deterministic request, the Request-Hash option is treated as a Class I option. This creates the request-response binding ensuring that no mismatched responses can be successfully unprotected.
The option does not actually need to be present in the message as transported (the server SHOULD elide it for compactness).
The client MUST replace any Request-Hash values present in the response with the Request-Hash it sent in the request before any OSCORE processing.

<!--
MT: Is there any possible reason in this application of the Request-Hash option to not elide it the from the response?
-->

\[ Suggestion for any OSCORE v2: avoid request details in the request's AAD as individual elements. Rather than having 'request_kid', 'request_piv' and (in Group OSCORE) 'request_kid_context' as separate fields, they can better be something more pluggable.
This would avoid the need to make up an option before processing.  \]

When preparing the response, the server performs the following actions.

* The server sets a non-zero Max-Age option, thus making the Deterministic Request usable for the proxy cache.

* The server preliminarily sets the Request-Hash option with the full request hash.

* The server MUST protect the response using the group mode of Group OSCORE, as defined in {{Section 8.3 of I-D.ietf-core-oscore-groupcomm}}. This is required to ensure that the client can verify source authentication of the response, since the "pairwise" key used for the Deterministic Request is actually shared among all the group members.

  Note that the Request-Hash option is treated as Class I here.

* The server MUST use its own Sender Sequence Number as Partial IV to protect the response, and include it as Partial IV in the OSCORE option of the response. This is required since the server does not perform replay protection on the Deterministic Request (see {{ssec-use-deterministic-requests-response}}).

* The server uses 2.05 (Content) as outer code even though it is not necessarily an Observe notification {{RFC7641}}, in order to make the response cacheable.

* The server SHOULD remove the Request-Hash option from the message before sending.

Upon receiving the response, the client performs the following actions.

* In case the response includes a 'kid' in the OSCORE option and unless responses from multiple servers are expected (see {{det-req-one-to-many}}), the client MUST verify it to be exactly the 'kid' of the server to which the Deterministic Request was sent.

* The client removes any Request-Hash options from the response, and inserts a Request-Hash option with the full request hash in their place.

* The client verifies the response using the group mode of Group OSCORE, as defined in {{Section 8.4 of I-D.ietf-core-oscore-groupcomm}}. In particular, the client verifies the counter signature in the response, based on the 'kid' of the server it sent the request to. When verifying the response, the Request-Hash option is treated as a Class I option.

### Deterministic Requests to Multiple Servers ### {#det-req-one-to-many}

A Deterministic Request *can* be sent to a CoAP group, e.g. over UDP and IP multicast {{I-D.ietf-core-groupcomm-bis}}, thus targeting multiple servers at once.

To simplify key derivation, such a Deterministic Request is still created in the same way as a one-to-one request and still protected with the pairwise mode of Group OSCORE, as defined in {{sssec-use-deterministic-requests-client-req}}.

\[ Note: If it was protected with the group mode, the request hash would need to be fed into the group key derivation just for this corner case. Furthermore, there would need to be a signature from the absent public key. \]

When a server receives a request from the Deterministic Client as addressed to a CoAP group, the server proceeds as defined in {{sssec-use-deterministic-requests-server-req}}, with the difference that it MUST include its own Sender ID in the response, as 'kid' parameter of the OSCORE option.

Although it is normally optional for the server to include its Sender ID when replying to a request protected in pairwise mode, it is required in this case for allowing the client to retrieve the Recipient Context associated to the server originating the response.

# Security Considerations # {#sec-security-considerations}

The same security considerations from {{RFC7252}}{{I-D.ietf-core-groupcomm-bis}}{{RFC8613}}{{I-D.ietf-core-oscore-groupcomm}} hold for this document.

Compared to Group OSCORE,
deterministic requests dispense with some of OSCORE's security properties
by just so much as to make caching possible:

* Receiving a response to a deterministic request does not mean that the response was generated after the request was sent.

  It still contains two freshness statements, though:

  * It is more recent than any other response from the same group member that has a smaller sequence number.
  * It is more recent than the original creation of the deterministic security context.

<!--
MT: "more recent than the original creation ..." By whom? Perhaps it means: "It is more recent than any deterministic request protected by the same Deterministic Client." ?
-->
  
* Request confidentiality is limited.

  An intermediary can determine that two requests from different clients
  are identical, and associate the different responses generated for them.
  Padding is suggested for responses where necessary.

<!--
MT: On the second sentence, does this mean to possibly use the Padding CoAP option defined in Appendix B, or whatever else padding mechanism?
-->
  
* Source authentication for requests is lost.

  Instead, the server must verify that the request (precisely: its handler) is side effect free.
  The distinct semantics of the CoAP request codes can help the server make that assessment.

\[ More on the verification of the Deterministic Request \]

# IANA Considerations # {#iana}

This document has the following actions for IANA.

## CoAP Option Numbers Registry ## {#iana-coap-options}

IANA is asked to enter the following option numbers to the "CoAP Option Numbers" registry defined in {{RFC7252}} within the "CoRE Parameters" registry.

~~~~~~~~~~~
+--------+--------------+-------------------+
| Number |     Name     |     Reference     |
+--------+--------------+-------------------+
|  TBD1  | Request-Hash | [[this document]] |
+--------+--------------+-------------------+
|  TBD2  | Padding      | [[this document]] |
+--------+--------------+-------------------+
~~~~~~~~~~~
{: title="CoAP Option Numbers" artwork-align="center"}

\[

For the Request-Hash option, the number suggested to IANA is 548.

For the Padding option, the option number is picked to be the highest number in the Experts Review range; the high option number allows it to follow practically all other options, and thus to be set when the final unpadded message length including all options is known. Therefore, the number suggested to IANA is 64988.

Applications that make use of the "Experimental use" range and want to preserve that property are invited to pick the largest suitable experimental number (65532)

Note that unless other high options are used, this means that padding a message adds an overhead of at least 3 bytes, i.e. 1 byte for option delta/length and two more bytes of extended option delta. This is considered acceptable overhead, given that the application has already chosen to prefer the privacy gains of padding over wire transfer length.

\]

--- back

# Change log

Since -01:

* Not meddling with request_kid any more.

  Instead, Request-Hash in responses is treated as Class I, but typically elided.

  In requests, this removes the need to compute the external_aad twice.

* Derivation of the hash now uses the external_aad, rather than the full AAD. This is good enough because AAD is a function only of the external_aad, and the external_aad is easier to get your hands on if COSE manages all the rest.

* Alignment with changes in core-oscore-groupcomm-12.

* Editorial improvements.

Since -00:

* More precise specification of the hashing (guided by first implementations)

* Focus shifted to deterministic requests (where it should have been in the first place; all the build-up of Token Requests was moved to a motivating appendix)

* Aligned with draft-tiloca-core-observe-responses-multicast-05 (not submitted at the time of submission)

* List the security properties lost compared to OSCORE

# Padding {#sec-padding}

As discussed in {{sec-security-considerations}}, information can be leaked by the length of a response or, in different contexts, of a request.

In order to hide such information and mitigate the impact on privacy, the following Padding option is defined, to allow increasing a message's length without changing its meaning.

The option can be used with any CoAP transport, but is especially useful with OSCORE as that does not provide any padding of its own.

Before choosing to pad a message by using the Padding option, application designers should consider whether they can arrange for common message variants to have the same length by picking a suitable content representation; the canonical example here is expressing "yes" and "no" with "y" and "n", respectively.

<!--
MT: Can we elaborate more on how the Padding option makes it more difficult for an adversary to relate responses to two different clients? After all, the Padding option is well visible and recognizable.
-->

## Definition of the Padding Option

As summarized in {{padding-table}}, the Padding option is elective, safe to forward and not part of the cache key; these follow from the usage instructions. The option may be repeated, as that may be the only way to achieve a certain total length for the padded message.

~~~
+------+---+---+---+---+---------+--------+--------+---------+
| No.  | C | U | N | R |  Name   | Format | Length | Default |
+------+---+---+---+---+---------+--------+--------+---------+
| TBD2 |   |   | x | x | Padding | opaque | any    | (none)  |
+------+---+---+---+---+---------+--------+--------+---------+
~~~
{: #padding-table title="Padding Option" artwork-align="center"}

## Using and processing the Padding option

A client may set the Padding option, specifying any content of any length as its value.

A server MUST ignore the option.

Proxies are free to keep the Padding option on a message, to remove it or to add further padding of their own.

# Simple Cacheability using Ticket Requests {#sec-ticket-requests}

Building on the concept of Phantom Requests and Informative Responses defined in {{I-D.ietf-core-observe-multicast-notifications}},
basic caching is already possible without building a Deterministic Request.

The approach discussed in this appendix is not provided for application. In fact, it is efficient only when dealing with very large representations and no OSCORE inner Block-Wise mode (which is inefficient for other reasons), or when dealing with observe notifications (which are already well covered in {{I-D.ietf-core-observe-multicast-notifications}}).

Rather, it is more provided as a "mental exercise" for the authors and interested readers to bridge the gap between this document and {{I-D.ietf-core-observe-multicast-notifications}}.

That is, instead of replying to a client with a regular response, a server can send an Informative Response, defined as a protected 5.03 (Service Unavailable) error message. The payload of the Informative Response contains the Phantom Request, which is a Ticket Request in this document's broader terminology.

Unlike a Deterministic Request, a Phantom Request is protected in Group Mode.
Instead of verifying a hash, the client can see from the signature that this was indeed the request the server is answering.
The client also verifies that the request URI is identical between the original request and the Ticket Request.

The remaining exchange largely plays out like in {{I-D.ietf-core-observe-multicast-notifications}}'s "Example with a Proxy and Group OSCORE":
The client sends the Phantom Request to the proxy (but, lacking a ``tp_info``, without a Listen-To-Multicast-Responses option),
which forwards it to the server for lack of the option.

The server then produces a regular response and includes a non-zero Max-Age option as an outer CoAP option. Note that there is no point in including an inner Max-Age option, as the client could not pin it in time.

When a second, different client later asks for the same resource at the same server, its new request uses a different 'kid' and 'Partial IV' than the first client's. Thus, the new request produces a cache miss at the proxy and is forwarded to the server, which responds with the same Ticket Request provided to the first client. After that, when the second client sends the Ticket Request, a cache hit at the proxy will be produced, and the Ticket Request can be served from the proxy's cache.

When multiple proxies are in use, or the response has expired from the proxy's cache, the server receives the Ticket Request multiple times. It is a matter of perspective whether the server treats that as an acceptable replay (given that this whole mechansim only makes sense on requests free of side effects), or whether it is conceptualized as having an internal proxy where the request produces a cache hit.

# Application for More Efficient End-to-End Protected Multicast Notifications {#det-requests-for-notif}

{{I-D.ietf-core-observe-multicast-notifications}} defines how a CoAP server can serve all clients observing a same resource at once, by sending notifications over multicast. The approach supports the possible presence of intermediaries such as proxies, also if Group OSCORE is used to protect notifications end-to-end.

However, comparing the "Example with a Proxy" in {{Section A of I-D.ietf-core-observe-multicast-notifications}} and the "Example with a Proxy and Group OSCORE" in {{Section B of I-D.ietf-core-observe-multicast-notifications}} shows that, when using Group OSCORE, more requests need to hit the server. This is because every client originally protects its Observation request individually, and thus needs a custom response served to obtain the Phantom Request as a Ticket Request.

If the clients send their requests as the same deterministic request, the server can use these requests as Ticket Requests as well. Thus, there is no need for the server to provide a same Phantom Request to each client.

Instead, the server can send a single unprotected Informative Response - very much like in the example without Group OSCORE - hence setting the proxy up and optionally providing also the latest notification along the way.

The proxy can thus be configured by the server following the first request from the clients, after which it has an active observation and a fresh cache entry in time for the second client to arrive.

# Open questions

* Is "deterministic encryption" something worthwhile to consider in COSE?

  COSE would probably specify something more elaborate for the KDF
  (the current KDF round is the pairwise mode's;
  COSE would probably run through KDF with a KDF context structure).

  COSE would give a header parameter name to the Request-Hash
  (which for the purpose of OSCORE deterministic requests would put back into Request-Hash by extending the option compression function across the two options).

  Conceptually, they should align well, and the implementation changes are likely limited to how the KDF is run.

* An unprotection failure from a mismatched hash will not be part of the ideally constant-time code paths that otherwise lead to AEAD unprotect failures. Is that a problem?

  After all, it does tell the attacker that they did succeed in producing a valid MAC (it's just not doing it any good, because this key is only used for deterministic requests and thus also needs to pass the Request-Hash check).

<!--
MT: This second bullet point seems something that can already be said in the Security Considerations section.
-->
  
# Unsorted further ideas

* All or none of the deterministic requests should have an inner observe option.
  Preferably none -- that makes messages shorter, and clients need to ignore that option either way when checking whether a Consensus Request matches their intended request.

<!--
MT: How can clients start an observation then? That would require an inner Observe option, right?

Also the guidelines in Section 2 suggest to have an inner observe option, regardless the resource being actually observable.
-->
  
# Acknowledgments # {#acknowldegment}
{: numbered="no"}

The authors sincerely thank Michael Richardson, Jim Schaad and Göran Selander for their comments and feedback.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
