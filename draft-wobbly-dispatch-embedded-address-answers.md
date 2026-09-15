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

Almost all web interactions begin with resolving a name to an IP address using
the DNS. While the DNS is performant, especially when the locals resolver's
cache has the answer, there is still a measurable delay and performance impact.
In addition to the performance impact, there is also a privacy implication in
sharing the name being looked up with a third party to the transaction (the
DNS).

The mechanisms described in this document provide means to provide resolution
without (directly) asking the DNS.


--- middle

# Introduction

This document provides mechanisms to embed resolved IP addresses in web
interactions, thereby improving performance, as well as potentially improving
privacy. It contains a number of mechanisms for embedding or transporting these
answers, as well as multiple mechanisms for using these embedded answers.

{Ed note: The authors of this document are more "DNS people" than "web people".
While we have discussed these with "web people" we are certainly not experts
and kindly ask readers make accommodations for our clunky terminology, etc.  }

Take, for example, a user loading www.example.com/kittens.html. This webpage
contains a number of links to resources, including an image of a kitten at
images.example.com, and a link to another page, www.example.net/bunnies.html.

In order to load the image, the user's browser needs to resolve
images.example.com, and if the user wants to follow the link to learn about
bunnies, they will also need the address for www.example.net. Both of these
require performing DNS lookups, which both take time, and also leak the fact
that the user is getting an image, and also (probably) is interested in
bunnies. While this example is not privacy sensitive, because everyone likes
bunnies and kittens, there are obviously situations where the user might not
want what they are resolving exposed to their DNS servers. Also, while there
are methods to decrease the latency from doing DNS resolutions, such as
prefetching and rendering while fetching additional resources, these are not
completely effective.

By embedding the resolution targets (generally IP address records) in the
response (see below for possible embeddings), we can both improve user privacy
and experience.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Yeah, sez you!

The obvious question is why a client would be willing to trust an embedded
response. This section discusses a some options.

{Ed note: This section is intended to drive (hopefully robust but congenial!)
discussion. }

## DNSSEC

The most obvious (and probably more secure) reason that an embedded answer
could be trusted is if the domain is DNSSEC signed, and the full set of
resources records is included in the embedding. This allows the browser to
cryptographically validate that the answer is one intended by the zone
operator.

This obviously has some limitations - the primary one being that the domain
needs to be DNSSEC signed!

## Connection racing

A second option is not to actually trust the embedded answer, and instead use
it simply as an optimization hint. HTTP Connection racing, as described in
[TODO] allows a browser to begin making a connection to a server, but not
actually complete that connection until it has received additional
confirmation.

In this proposal, the browser could begin making a "raced" connection to the
embedded address, possibly including the TLS handshake, etc but not actually
send any data until it has also validated the address through DNS. In this
design, the browser would strictly treat the embedded address as not trusted
until it has received an identical answer through the DNS - but it can at
improve the users' experience by having begun a connection to that server.

{Ed note: This might be a bad idea, more discussion is needed. DNS-based load
balancing (sometimes called GSLB) used to be very common, and so the addresses
that clients received were fairly dynamic, but increasingly CDNs are moving to
anycast, and non-CDN services generally only have a small number of IPs.  }

{Ed note: As a web page already has multiple ways to cause a connection to be
made to arbitrary places (e.g embedding resources, Javascript, etc), this
doesn't seem to really create additional capabilities for malicious pages, but
it's entirely possible that we've missed something obvious!}


## "Same origin / same domain"

{Ed note: Hey, we did note that our web-terminology is shaky.} {Ed note: This
might be the worst idea ever. Seriously, it might be, but we wanted some
feedback if there is a way to make it less bad...}

It is very likely that www.example.com and images.example.com are operated by
the same entity, and that it might be safe to fetch content from the address
for images.example.com embedded in www.example.com, subject to certain
restrictions -- for example, the answer would need to be in the same "domain",
the answer could only be used to fetch (not post) content, etc.

It's entirely possible that this is a really stupid idea, but we want some
feedback - for example, www.example.com embedding an answer for
images.example.com "feels" okay, but the same thing is not true for
example1.medium.com and example2.medium.com. Something something PSL something
same-origin something?


# Embedding options

This document primarily discusses the concepts around embedded answers, and
these are some potential mechanisms to embed these answers in connections. As
noted above, the primary authors are "DNS people".

## HTML

An obvious, naive mechanism would be to simply embed the resolved IP address in
the HTML itself. This is likely the least optimal was of accomplishing the
goals expressed in this document, and is primarily shown here as an
illustrative example (we don't really think it will survive adoption!).

There are obviously many potential issues with this approach, including stale
answers, but it could potentially be useful if the HTML were dynamically
generated - but again, this is primarily documented to illustrate the concept.

The page at www.example.com/kittens could contain tags such as:
~~~
<img src="https://images.example.com/kittens.png" address="192.0.2.1"></img>
<a href="https://www.example.com/bunnies.html" address="192.0.2.1,192.0.2.42,[TODO]">
~~~

An advantage of this approach (other than it being a good illustration of the
concept!) is that it is trivial for a website or [TODO] to implement. If this
is actually deployed, the authors think that it would only be for testing /
experimentation. It does, however, put the webpage author in charge.

## HTTP Header

In this embedding mechanism, the embedded answers would be carries in an HTTP
Header. For example:

~~~
Embedded-Answers="images.example.com: [192.0.2.1], www.example.net: [192.0.2.1, 192.0.2.2, ]
~~~

Note that this is also simply an illustrative example, the format, name, etc
would obviously need to be changed after some discussion with a working group.

This has the advantage of being architecturally cleaner, but does require that
the webserver implementation embed the answers. This might be a good option, as
in many cases the web-server will have good visibility into what the "best"
answer is for a client. For example, it is likely that the same web-servers or
CDN serves both www.example.com and images.example.com. This means that the
webserver for www.example.com (potentially) has good visibility into the
relative load, capabilities, cache, etc for images.example.com.

This solution places the responsibility for resolving and embedding the answers
at the webserver / infrastructure level. This obviously has some implications -
for example, if the site owner has contracted with multiple CDNs, the CDN
operator may have an incentive to preferentially return their own IP addresses
to retain more traffic, or conversely return their competitors's addresses to
drive costs to them.

## QUIC / HTTP/3 streams.

{ Someone who actually understands this stuff will need to write this section
:-) }

# Resolving the address / implications.

Obviously, before an answer can be embedded in a response, the entity
generating the response will need to determine what that answer should be. The
"correct" answer depends on many factors - as an example, a CDN serving
resources for Customer A may serve multiple names for that customer, and
"internal" links are quite common. It is also possible that the same CDN might
be serving other customers that Customer A includes resource from, or links to.

If this is not the case, the webserver or CDN could proactively resolve names
using the DNS, which it knows that clients will need. There are a few obvious
tensions here:
  1. how often should the CDN resolve these names? There is a tradeoff between
     creating excess load resolving names which are not needed versus the
     resolutions that the clients do not need to make because they have been
     given the answer. Determining when and how often to resolve this is a
     topic for discussion. although DNS based load-balancing is decreasing in
     popularity, it is still in common use. If a CDN in e.g Toronto Canada
     resolves www.example.net it may receive an IP address which is
     network-topographically "close" to Toronto, which may be completely
     unsuitable for a client in e.g Sydney Australia.
  3. TTLs and load-sloshing: If a large CDN resolves a popular name like
     www.example.com and hands that resolved IP to all of its clients, that IP
     may experience significant load. If the next time that CDN resolves the
     name it gets a different answer, and then starts using that, there might
     be large, and unwelcome traffic shifts.

# Security Considerations

Yes, we are sure that there are many security considerations here. This entire
idea might be a really bad one. The authors think that this idea is worth
exploring, but we are also fine to retitle this "Embedded Answers Considered
Dangerous" and document why :-)


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Remember to update this. Initial list: David Schinazi, Erik Nygren, everyone
involved in "Resolverless DNS", Kraftwerk,
