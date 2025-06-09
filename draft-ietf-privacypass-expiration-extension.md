---
title: "Privacy Pass Token Expiration Extension"
abbrev: "Privacy Pass Token Expiration Extension"
category: std

docname: draft-ietf-privacypass-expiration-extension-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Privacy Pass"
keyword:
 - token
 - extensions
venue:
  group: "Privacy Pass"
  type: "Working Group"
  mail: "privacy-pass@ietf.org"
  github: "chris-wood/draft-ietf-privacypass-expiration-extension"

author:
 -
    fullname: Scott Hendrickson
    organization: Google
    email: "scott@shendrickson.com"
 -
    fullname: Christopher A. Wood
    organization: Cloudflare, Inc.
    email: caw@heapingbits.net

normative:
   AUTH-EXTENSIONS: I-D.wood-privacypass-auth-scheme-extensions
   AUTH-SCHEME: I-D.ietf-privacypass-auth-scheme
   EXTENDED-ISSUANCE: I-D.ietf-privacypass-public-metadata-issuance
   BASIC-ISSUANCE: I-D.ietf-privacypass-protocol
   ARCHITECTURE: I-D.ietf-privacypass-architecture
   CONSISTENCY: I-D.ietf-privacypass-key-consistency


--- abstract

This document describes an extension for Privacy Pass that allows tokens
to encode expiration information.

--- middle

# Introduction

Some Privacy Pass token types support binding additional information to
the tokens, often referred to as public metadata. {{AUTH-EXTENSIONS}} describes
an extension parameter to the basic PrivateToken HTTP authentication scheme {{AUTH-SCHEME}}
for supplying this metadata alongside a token. {{EXTENDED-ISSUANCE}} describes
variants of the basic Privacy Pass issuance protocols {{BASIC-ISSUANCE}} that
support issuing tokens with public metadata.

This document describes an extension for Privacy Pass that allows tokens
to encode expiration information. The use case and deployment considerations,
especially with respect to the resulting privacy impact, are also discussed.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Expiration Extension

The expiration extension is used to convey the expiration for an issued token.
It is useful for Privacy Pass deployments that make use of cached tokens, i.e.,
those that are not bound to a specific TokenChallenge redemption context, when
they want to limit token lifetime without having to frequently rotate issuing
key pairs.

For example, consider a Privacy Pass deployment wherein Clients use cached tokens that
are valid for one hour. Clients could pre-fetch these tokens each hour and the Issuer
and Origin could rotate the verification key every hour to force expiration. Alternatively,
Clients could pre-fetch tokens for the entire day all at once, including an expiration
timestamp in each token to indicate the time window for which the token is valid.

The ExtensionType (defined in {{AUTH-SCHEME}}) of this extension is `0x01`. The
value of this extension is an ExpirationTimestamp, defined as follows.

~~~
struct {
   uint64 timestamp_precision;
   uint64 timestamp;
} ExpirationTimestamp;
~~~

The ExpirationTimestamp fields are defined as follows:

- "timestamp_precision" is an 8-octet integer, in network byte order, representing the granularity of the timestamp,
  i.e., the target to which the timestamp is rounded for loss of precision.

- "timestamp" is an 8-octet integer, in network byte order, representing the expiration timestamp. The
  expiration timestamp is the UNIX time in seconds at which a token expires, expressed in the UTC/GMT timezone ({{!RFC3339, Section 4.1}}).

As an example, an ExpirationTimestamp structure with the following value would be interpreted as an
expiration timestamp of 1688583600, i.e., July 05, 2023 at 19:00:00 GMT+0000, which is the timestamp
rounded to the nearest hour (timestamp_precision = 3600).

~~~
struct {
   uint64 timestamp_precision = 3600;
   uint64 timestamp = 1688583600;
} ExpirationTimestamp;
~~~

# Privacy Considerations {#privacy}

This extension intentionally adds more information to a token that might not otherwise be
visibile to Attester, Issuer, or Origin. As such, how this information is chosen can have
an impact on Origin-Client, Issuer-Client, Attester-Origin, or redemption context unlinkability
as defined in {{Section 3.2 of ARCHITECTURE}}. Mitigating risk of privacy violation requires
that the extension be constructed in a way that does not induce anonymity set partitioning,
as described in {{Section 6.1 of ARCHITECTURE}}.

The best way to achieve this in practice is for Clients to use the same limited sets of information
in the extension. Consistency can be achieved in a variety of ways. For example,
Client implementations might insist that all Clients use the same deterministic function for
computing the expiration timestamp, e.g., some function F(current time). This
function would round the current timestamp, resulting in a loss of precision but overall
less unique value. One way to implement this function would by rounding the timestamp
to the nearest hour, day, or week. Of course, this does not account for clock skew,
which occurs with some non-neglgiible probability in practice {{?CLOCK-SKEW=DOI.10.1145/3133956.3134007}}.

An alternative implementation strategy for consistency is to run some sort of consistency
check to ensure that the Client uses a value that is consistent with other Clients. Several
consistency mechanisms exist; see {{CONSISTENCY}} for more information. Such an explicit
consistency check would depend less upon the Client's current clock and thus be more robust
at the cost of additional work.

Orthogonal to the mechanism used to ensure consistency, it is also important that Clients
choose expiration timestamps that are shared by other Clients. Consider, for example, a
scenario where two Clients consistently choose expiration timestamps per the recommendation
above, but only one Client ever requests a token within a given expiration window. Despite
the consistency check in place, the actual value of the timestamp is still unique to one of
the Clients.

The means by which implementations ensure that some minimum number of Clients share the same
expiration timestamp is a deployment-specific challenge. For example, in the Split Origin,
Attester, and Issuer deployments as described in {{Section 4.4 of ARCHITECTURE}}, the Attester
is positioned to ensure that Clients do not choose consistent yet unique values. General purpose
approaches to ensure that some minimum number of Clients share the same expiration timestamp
are outside the scope of this document; indeed, this problem is not unique to Privacy Pass and
is common to other privacy-related protocols such as Oblivious HTTP {{?OHTTP=I-D.ietf-ohai-oblivious-http}}.

# Security Considerations

Use of the expiration extension risks revealing additional information to parties that see the
extension, including the Attester, Issuer, and Origin. {{privacy}} discusses specific privacy
implications for use of this extension that aim to mitigate exposure of information that can
unintentionally partition the Client anonymity set and lead to Origin-Client, Issuer-Client,
Attester-Origin, or redemption context unlinkability as defined in {{Section 3.2 of ARCHITECTURE}}.
General information regarding the use of extensions and their possible impact on Client privacy
can be found in {{Section 3.4.3 of ARCHITECTURE}} and {{Section 6.1 of ARCHITECTURE}}.

# IANA Considerations

This document registers the following entry into the "Privacy Pass PrivateToken Extensions" registry.

- Expiration extension
   - Type: 0x0001
   - Name: Expiration
   - Value: ExpirationTimestamp value as defined in {{expiration-extension}}
   - Reference: This document
   - Notes: None

--- back

# Acknowledgments
{:numbered="false"}

This document received input and feedback from Jim Laskey.
