---

title: "DNS Embedded Answers"
abbrev: "DAE"
category: info

docname: draft-wobbly-dispatch-embedded-address-answers-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - address resolution
 - dns
 - performance
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
  -
    fullname: Warren Kumari
    organization: Google, Inc.
    email: warren@kumari.net
  -
    fullname: Joe Abley
    organization: Cloudflare
    email: jabley@cloudflare.com


normative:

informative:

...

--- abstract

Many web interactions require a client to resolve a hostname before it can
establish a connection to a service. While DNS is generally efficient, each
lookup incurs some delay and exposes information about the user's browsing
activity to the recursive resolver and upstream infrastructure. This document
explores techniques by which a client may receive DNS-derived address data as
part of an application-layer response, so that it can initiate a connection
without performing a separate DNS lookup for the same target.

The mechanisms discussed here are intended to reduce lookup latency and
potentially improve privacy, but they also raise significant questions about
trust, freshness, and security. The document does not prescribe a single
normative mechanism; instead, it describes several design options and the
trade-offs associated with each.

--- middle

# Introduction

The vast majority of web interactions begin with a hostname resolution step. A
browser that needs to fetch an image from images.example.com or follow a link
to www.example.net typically performs one or more DNS lookups before opening a
connection. Even when a resolver cache produces a fast answer, the lookup can
still impose measurable latency and can reveal information about the user's
activities to the resolver and any intermediaries on the path.

This document discusses a class of mechanisms in which the client receives
address information as part of an existing application-layer exchange, rather
than by performing a separate DNS lookup. Such information could be embedded in
HTML, carried in HTTP metadata, or delivered through a protocol mechanism such
as HTTP/3 or QUIC. The goal is to allow the client to establish a connection
more quickly, and in some cases to avoid exposing the lookup itself to the DNS
infrastructure.

This document is intentionally exploratory. It presents a number of possible
ways to embed address information, along with several trust models that a
client might apply when deciding whether to use such information. The design
space is broad, and the trade-offs are substantial.

Consider a page at https://www.example.com/kittens.html that includes an image
served from https://images.example.com/kittens.png and a link to
https://www.example.net/bunnies.html. To fetch the image, the browser must
resolve images.example.com. To follow the link, it must resolve
www.example.net. Both operations take time and reveal that the user is
accessing those resources. Prefetching and speculative connection setup can
reduce the delay, but they do not eliminate it and they do not necessarily
remove the privacy implications of host resolution.

If the origin or a trusted intermediary were able to provide the relevant
address information in the response itself, the client could potentially avoid
a separate lookup or begin connection setup earlier. This could improve the
user experience while reducing exposure of the requested hostnames to third
parties.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the term "embedded answer" to refer to an application-layer
value that conveys one or more network addresses associated with a hostname.
The term is intentionally generic: it does not assume a specific format or a
single authority model.

The term "optimization hint" refers to a value that may improve connection
setup latency, but is not necessarily authoritative. The term "authenticated
answer" refers to a value that the client has reason to treat as equivalent to
an answer obtained through DNS validation or other cryptographic means.

# Goals and Non-Goals

The goals of the techniques discussed here are to:

- reduce the latency incurred by hostname resolution;
- allow a client to begin connection setup earlier; and
- reduce the amount of hostname information exposed to the DNS path.

These techniques are not limited to a single transport or a single application
protocol. However, the examples in this document focus on web content and
connection establishment.

The non-goals include:

- replacing DNS as the primary name resolution system;
- guaranteeing that embedded answers are universally trusted; or
- defining a single protocol mechanism without further discussion and
  deployment experience.

# Trust Models

A central question is why a client would trust an embedded answer. This section
outlines several possible trust models, each with different security and
operational properties.

## DNSSEC-validated answers

The strongest model is one in which the embedded answer is cryptographically
bound to the originating DNS zone. For example, a response could include the
relevant RRset, along with the necessary DNSSEC validation material, and the
client could verify the answer using the DNSSEC chain. If the validation
succeeds, the answer is equivalent to an authenticated DNS answer.

This model has the advantage that it aligns closely with the existing security
properties of DNS. It does, however, require that the relevant zone be DNSSEC
signed and that the client and the embedding entity support the necessary
validation steps.

## Optimization-only hints

A second model is to treat the embedded answer as an optimization hint rather
than as authoritative data. In such a model, the client may use the address to
begin connection setup or perform connection racing while waiting for DNS to
confirm or refute the advertised address.

In this design, the client does not necessarily consider the embedded answer to
be trusted until it has been validated by a conventional DNS resolution or by
another trusted mechanism. The benefit is that the client can exploit the
embedded data to reduce connection setup latency without accepting the risk of
blindly trusting it.

This model may be useful when the embedded answer is generated by a local
infrastructure component that is in a position to estimate the best endpoint,
but it still preserves a fallback path to DNS-driven verification.

## Same-origin or same-operator assumptions

A third model is to treat addresses associated with a related hostname as safe
to use within a constrained trust boundary, such as a site or an organization.
For example, a page at www.example.com might be allowed to carry an embedded
answer for images.example.com if the hostnames are controlled by the same party
and the client applies explicit restrictions.

This model is attractive because it aligns with common web deployment patterns,
but it is also more dangerous. It depends on an operational assumption that is
not necessarily true even when the hostnames share a registrable domain. The
client must define restrictions such as whether the value may be used only for
read operations, only for a limited set of protocols, or only for a specific
origin relation. It must also consider the possibility that a page can cause a
client to connect to destinations that are not obviously related to the origin.

The precise boundaries for "same-origin" and "same-operator" are not obvious,
and this remains a topic for further discussion.

# Embedding Mechanisms

This document discusses several possible ways to deliver embedded answers. The
purpose of this section is to illustrate the design space, not to mandate a
single mechanism.

## HTML embedding

A straightforward design is to embed the address information directly in HTML.
For example, the page at https://www.example.com/kittens could include
attributes on elements that indicate the network target for a given resource:

~~~ html
<img src="https://images.example.com/kittens.png"
     address="192.0.2.1" />

<a href="https://www.example.net/bunnies.html"
   address="192.0.2.1, 192.0.2.42, 2001:0DB8::42">Bunnies! Yay, bunnies!</a>
~~~

This mechanism is simple to understand and easy to prototype. It has the
advantage that the page author or generation system controls the information
that is embedded. It also has obvious drawbacks: the information may become
stale, the browser must interpret a potentially large number of address hints,
and the page author could embed values that are not aligned with the current
DNS or the client's network conditions.

The approach is useful for experiments, but it is not likely to be the final
preferred mechanism for general deployment unless the format is carefully
specified and tightly constrained.

## HTTP headers

A more structured design is to carry address information in an HTTP header.
This would allow the server, edge cache, or CDN to provide a set of answer
hints that are relevant to the response being served:

~~~
Embedded-Answers: images.example.com=192.0.2.1; www.example.net=192.0.2.1,192.0.2.2
~~~

This mechanism is architecturally cleaner than embedding data directly in HTML.
The server is already in control of the response, and in many deployments it
has valuable context about origin topology, load, and client affinity. In some
cases, the same server or CDN is responsible for both the referencing page and
the target resource, which makes it easier to select an appropriate endpoint.

The drawback is that the server becomes responsible for resolving and selecting
addresses. This introduces incentives and operational trade-offs. For example,
a CDN that fronts multiple origins might prefer a local address or a preferred
peer network in order to retain traffic, potentially at the expense of the
client's optimal path. This is not necessarily an error in the protocol design,
but it is a substantial deployment consideration.

## QUIC and HTTP/3 transport mechanisms

A protocol-native mechanism could allow the client to receive address hints as
part of an HTTP/3 or QUIC exchange. Such a mechanism might be integrated into
the transport context, a response header, or an extension-specific metadata
field. This approach may be more efficient in environments where the transport
and applications are tightly coupled, but it requires a specific protocol
design and a clear trust model.

This document does not attempt to define such a format in detail. It only notes
that transport-specific mechanisms are likely to offer efficiency advantages,
particularly when the client can apply the embedded data in a way that is
consistent with the underlying connection establishment logic.

# Address Selection and Operational Considerations

Before an answer can be embedded in a response, the entity generating the
response must determine what address information to provide. In many cases, the
correct answer depends on factors such as:

- the client's local network conditions;
- the target service's current load and endpoint set;
- the transport and policy of the origin or CDN; and
- the service's DNS configuration and load-balancing behavior.

For example, a CDN serving multiple customers may have different optimal
endpoints for the same hostname depending on the client network, the edge
location, and the active service topology. A server that resolves a name in one
geographic region may produce an answer that is excellent for Toronto clients
but poor for clients in Sydney.

This creates several operational tensions:

1. Resolution frequency: resolving names too often increases server-side load,
   while resolving too infrequently leads to stale answers and reduced
   effectiveness.
2. Load shifts: if a client receives an address that is correct for a previous
   mapping but not for the current service state, the answer may create an
   unintended traffic spike.
3. TTL semantics: the lifetime of the answer matters. A stale or overly broad
   answer can be worse than no answer at all.
4. Client heterogeneity: different clients may have distinct optimal
   destinations for the same hostname, which complicates any attempt to
   generate a single answer for all clients.

The design implications are therefore not purely protocol-level. They depend on
operator practices, client policies, and the operational realities of modern
CDN and edge infrastructure.

# Discussion Items

This document is intentionally exploratory, and the authors invite feedback on
a number of design questions. The purpose of this section is to highlight the
areas where the concept is still under discussion and where further experience
would be valuable.

## What is the appropriate trust boundary?

A key question is whether a client should treat embedded answers as
cryptographically authenticated, as a best-effort optimization hint, or as an
origin-local mechanism with explicit restrictions. The appropriate answer may
depend on the embedding mechanism, the application context, and the client's
security policy.

## What restrictions are necessary for cross-origin use?

A mechanism that carries address hints across origins introduces a question of
scope: should such hints be usable only for a single origin, only for related
hostnames, or only for restricted classes of requests? This question is closely
related to the broader web-security discussion of site, origin, and delegated
authority.

## How should freshness be managed?

Embedded addresses may become stale, either because the underlying endpoint set
changes or because the client receives an answer that was generated under older
conditions. The document does not yet specify how a client should bound the
lifetime of such an answer or how a server should determine when to refresh it.

## What is the right balance between privacy and performance?

The concept is intended to reduce DNS exposure and improve latency, but it also
creates a new set of data flows. It is not yet clear whether the privacy
benefit justifies the additional complexity or whether the answer should remain
limited to cases where the client can clearly describe the trust assumptions.

## What is the appropriate scope of deployment?

A significant open question is whether embedded answers should be deployed as a
browser-facing optimization, a server-side origin facility, or a
transport-level feature. Different deployment models may impose different
security, operational, and architectural constraints.

# Security and Privacy Considerations

This proposal creates a tension between optimization and trust. The same
mechanism that reduces DNS lookup latency can also allow a party that controls
a response to steer the client toward a chosen address without the client
having independently validated that address.

The security considerations include:

- stale or incorrect answers;
- traffic steering to a malicious or unintended endpoint;
- downgrade or bypass of DNSSEC validation;
- cross-origin trust assumptions;
- abuse by compromised origins or intermediaries; and
- privacy exposure from embedded data itself.

A client must therefore be conservative about how it uses embedded answers. The
safest model is to treat them as equivalent to DNS answers only when they are
cryptographically authenticated or otherwise validated according to a defined
trust policy. In the absence of such validation, the embedded value should be
handled as a hint and used only under constrained conditions.

Any design that permits embedded answers to alter a client's network connection
must specify the security boundary of that decision. In particular, the draft
must clearly explain whether the answer may be used for all requests, only for
certain origins, only for read-only traffic, or only in conjunction with a
separate DNS validation step.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the following individuals for their feedback
and suggestions: Mike Bishop, Geoff Huston, Erik Nygren, Adam Roach, David
Schinazi, Ian Swett. This work was informed by discussion on the "Resolverless
DNS" mailing list and other places, and we would like to acknowledge the
contributions of many individuals who have participated in these discussions
over the years.

Unfortunately, at least one of the authors has a terrible memory, and has lost
track of all those who have contributed to this topic over the years, and will
be more than happy to acknowledge their input if reminded of this :-)
