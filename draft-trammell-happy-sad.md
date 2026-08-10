---

title: "Slow Alternate Detection for Happy Eyeballs"
category: info

docname: draft-trammell-happy-sad-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Heuristics and Algorithms to Prioritize Protocol deploYment"
keyword:
 - icmp
 - happy eyeballs
 - debugging
 - measurement
venue:
  group: "Heuristics and Algorithms to Prioritize Protocol deploYment"
  type: "Working Group"
  mail: "happy@ietf.org"
  github: "britram/happy-sad"

author:
 -
    fullname: Brian Trammell
    organization: Google Switzerland GmbH
    email: ietf@trammell.ch
 -
    fullname: Suresh Krishnan
    email: suresh.krishnan@gmail.com

normative:

informative:

...

--- abstract

This document specifies Slow Alternate Detection (SAD) for Happy Eyeballs, an
ICMP-based advisory path signal {{?RFC8558}} for exposing information about path
non-selection on-path devices in order to aid debugging and measurement of Happy
Eyeballs.

--- middle

# Introduction

Happy Eyeballs {{!I-D.ietf-happy-happyeyeballs-v3}} encourages new protocol
deployment by reducing the availabity risk associated with attempting to use
them. However, in doing so, it masks configuration and deployment failures of
these very protocols. There are potential causes of such failures, with
potential root causes at the end user terminal, CPE, access network, CDN, DNS
configuration, and end server. Given the diffusion of root causes, debugging
these errors can be difficult.

This document presents Slow Alternate Detection (SAD), an ICMP-based advisory
path signal {{RFC8558}} to devices along the path of a non-selected candidate
designed as part of an array of approaches to this problem.  It is intended to
be used together with complementary monitoring and logging information at each
point along the potential failure chain of a non-selected candidate.

This design uses hashes of data relevant to a failure to allow the correlation
of nonselection events to on path devices and actors who already have access to
that data, while allowing only aggregate analysis by other on-path actors.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Message Format (ICMP)

The message format for SAD is identical for both ICMPv4 and ICMPv6,
and is depicted in {{fig-sad-message}}

~~~ artwork
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Type = 44   |     Code      | HAlg  |  ASR  |    NextHdr    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |    Source Transport Port      |  Destination Transport Port   |
   +-------------------------------+-------------------------------+
   |                     Additional Data                           |
   |                     (code-dependent)                          |
   ...                                                           ...
~~~
{: #fig-sad-message title="SAD Message Format"}

The Code, HAlg (Hash Algorithm), ASR (Approximate Sample Rate), Next Header and
Source and Destination Transport Port, and Additional Data fields are described
in the subsections below.

## Code

The ICMP Code for a SAD message takes one of the following values, and specifies
the message's semantics as well as the meaning of the Additional Data field. It
can take the following values:

| Code  | Description                      | Additional Data       |
|-------|----------------------------------|-----------------------|
|    0  | Not Selected                     | hashed DNS Answer     |
| 1-255 | Reserved                         | not present           |

### Not Selected

Not Selected indicates that is sent to a non-selected candidate by the client,
after it has made the decision not to use that candidate. See
{{not-selected-behavior}} for the use of this message.

When present, the Additional Data field of a Not Selected contains the DNS
Message {{!RFC1035}} associated with the answer that led to the connection attempt, hashed using the selected hash algorithm.

## Hash Algorithm (HAlg)

The Hash Algorithm field determines both the length of the Additional Data field
and the hash algorithm used to hash it. The following hash algorithms are
available:

| Value | Length | Algorithm                                                |
|-------|--------|----------------------------------------------------------|
|     0 |      0 | Additional Data Omitted                                  |
|     1 |    256 | SHA256 {{!RFC6234}}                                      |
|  2-15 |  undef | Reserved for Future Use                                  |

## Approximate Sample Rate {#approximate-sample-rate}

A client that reports only a sampled subset of its non-selected candidates (see
{{not-selected-behavior}}) uses the Approximate Sample Rate (ASR) field to expose
the rate at which it sampled, so that a receiver can estimate the number of
non-selection events that a given number of received messages represents. The
field takes one of the following values:

| Value | Sampling Rate             |
|-------|---------------------------|
|     0 | 1 in 1 (unsampled)        |
|     1 | 1 in 2                    |
|     2 | 1 in 5                    |
|     3 | 1 in 10                   |
|     4 | 1 in 20                   |
|     5 | 1 in 50                   |
|     6 | 1 in 100                  |
|     7 | 1 in 200                  |
|     8 | 1 in 500                  |
|     9 | 1 in 1,000                |
|    10 | 1 in 10,000               |
|    11 | 1 in 100,000              |
|    12 | 1 in 1,000,000            |
|    13 | 1 in 10,000,000           |
|    14 | Reserved for Future Use   |
|    15 | Sampled, rate undisclosed |

A client may sample its non-selected candidates at any rate it chooses, and
SHOULD set this field to the defined value nearest, in proportional terms, to the
rate it actually used. A receiver estimates the number of non-selection events
that a set of messages bearing a given value represents by multiplying their
count by the N in the value's "1 in N" rate. Because the reported value is the
nearest defined rate rather than the exact rate applied, this estimate is
approximate, with an error bounded by the spacing between adjacent defined rates;
the field is named accordingly.

The finer resolution at higher rates and coarser resolution at lower ones
reflects the expected use: an individual client sampling for its own reasons
operates at the higher rates, where the 1-2-5 spacing keeps the estimation error
of any single value small, while the decade steps at the low end serve
large-scale reporting deployments aggregating across many clients, for which a
coarser rate is sufficient.

A value of 0 indicates that the client is not sampling and reports every
non-selected candidate; each such message represents exactly one non-selection
event.

A value of 15 indicates that the client is sampling but does not disclose the
rate. A receiver MUST NOT use a message bearing this value to estimate a count of
non-selection events, as no unbiased estimate is possible from it; it MAY log or
count such messages as individual observations. This value allows a client to
treat its sample rate as sensitive, at the cost of the aggregate weighting its
messages would otherwise support; see {{sampling-as-a-privacy-control}}.

A client MUST NOT set this field to a value marked Reserved for Future Use. A
receiver that encounters a reserved value MUST treat it as it treats the value
15, and MUST NOT use the message for count estimation; a rate defined for a
reserved value by a later document is thereby ignored, rather than
misinterpreted, by a receiver implementing this document.

## 5-tuple fields

The Next Header (or Protocol) field contains the IP protocol (e.g. TCP, UDP) of
the non-selected path. The Source and Destination Transport Port fields contain
the source and destination port of the non-selected path. While this will not
allow NAT transparency of the SAD message, it does allow analysis and
correlation across the NAT by the operator thereof. If the candidate transport
protocol does not have port numbers, or if the client chooses not to expose
them, these fields are set to 0.

## Additional Data

The Additional Data contains code-specific additional data. If present, it MUST
be hashed with a hash algorithm specified by the Hash Algorithm field.

# Protocol Behaviors

Each of the (currently one) message code(s) associated with the SAD message has
an associated protocol behavior, defined below.

## Not Selected {#not-selected-behavior}

A client sends a Not Selected message after it has decided not to use a given
candidate identified by the 5-tuple in the Not Selected message. There is no
guarantee of the relative timing of the SAD Not Selected message and the
transport layer shutdown datagrams associated with this nonselection.

The client may send Not Selected messages on only a sampled portion of its
non-selected candidates. In this case, the client SHOULD expose the selected
sample rate in the Approximate Sample Rate field.

As this is an advisory path signal, forwarding elements and servers MUST NOT
take any action on the receipt of a Not Selected message beyond logging them for
later analysis.

In the following cases, the client MUST NOT include the hashed DNS answer in the
Not Selected message:

- if the DNS Answer message was retrieved from an off-path resolver
  (e.g., via DoH {{?RFC8484}}), since the answer would be useless to
  an on-path resolver.
- if the connection attempt used Encrypted Client Hello (ECH) {{?RFC9849}}
  in the TLS handshake, since the answer might not have sufficient entropy to
  allow the answer hash to adequately protect the server name.

# Security Considerations

As an explicit path signal, SAD exposes information about path non-selection to
devices along the path of a non-selected candidate, and does so in an
unauthenticated ICMP message carried outside any cryptographic context shared
with its receiver. The considerations in this section follow the guidelines in
{{?RFC3552}}, and assume an attacker that is either on the path between the
client and the non-selected candidate, in which case it can observe, delay,
drop, or modify SAD messages at will, or off the path, in which case it can
inject SAD messages provided it is able to spoof the source address of a client.
Neither capability is addressed by the mechanism described in this document.

## Message Injection and Spoofing

ICMP messages carry no source authentication, and SAD adds none. An off-path
attacker able to spoof a client's source address can therefore generate Not
Selected messages bearing an arbitrary five-tuple and arbitrary contents in the
Additional Data field, and a receiver of such a message has no means available
to it to distinguish that message from one generated by the client it purports
to come from.

The consequence of this is corruption of the datasets SAD exists to populate.
Since receivers are expected to log these messages for later analysis (see
{{not-selected-behavior}}), an attacker can inject apparent non-selection events
for candidates that were never offered, or in volumes that misrepresent the rate
at which non-selection occurs, and thereby bias aggregate analysis in a
direction of its choosing. A partial mitigation follows from the intended use of
the signal: since SAD is designed to be correlated with data the receiver
already holds, messages for which no corresponding local record exists can be
treated as unverified, and analyses that rest on SAD messages alone, without
such correlation, should be understood to rest on unauthenticated input.

## Use as a Traffic Direction Primitive

The destination address of a SAD message is chosen by the client, since it is by
definition the address of a candidate that the client has declined to use, and
nothing in this document constrains the set of addresses that a client may treat
as a candidate. A compromised client, or code running on an otherwise
well-behaved host, can therefore be made to emit Type 44 messages toward an
arbitrary victim address.

SAD provides no amplification: each message is generated by the client, elicits
no response, and is small, so the traffic that can be directed at a victim in
this way is bounded above by the traffic the compromised host could direct at
that victim in any case. The mechanism therefore adds no new denial-of-service
capability, though it may supply traffic of a kind that is less likely than
other traffic from the same host to be attributed to that host's compromise.
Ordinary ICMP rate limiting at forwarding elements, together with ingress
filtering {{?RFC2827}}, remains the appropriate response; SAD is not designed to
carry a defense of its own.

## Reaction to Advisory Signals

{{not-selected-behavior}} requires that receivers take no action on a Not
Selected message beyond logging it for later analysis. This is a design
constraint on receivers rather than a property of the protocol: it cannot be
enforced by the sender, and its observance cannot be verified by any party.

Where it is not observed, the injection capability described above becomes
considerably more consequential. A receiver that feeds SAD messages into
automated blocklisting, into traffic engineering decisions with immediate
effect, or into reconfiguration of server or resolver selection grants an
attacker able to spoof a client address a measure of direct influence over that
receiver's behavior, at a cost to the attacker of a single forged packet. The
restriction to logging is accordingly not merely a matter of layering hygiene in
the sense of {{RFC8558}}; it is the property that keeps a forgeable advisory
signal from becoming a control input.

## Interaction with Network Filtering

As a newly allocated ICMP type, SAD messages may be discarded by firewalls and
middleboxes configured to forward only those ICMP types known to them at the
time of their deployment. The pattern of transmission may attract attention
independently of the type allocation: messages bearing a fixed-length opaque
payload, sent to addresses with which the host has recently had short-lived
flows, resemble the traffic patterns of ICMP tunneling and of reconnaissance,
and may be treated as such by intrusion detection systems.

Delivery of SAD messages should therefore be expected to be unreliable in ways
that are neither uniform across paths nor stable over time. This is not harmful
to the client, whose operation does not depend on delivery, but it does mean
that the absence of SAD messages at a collector cannot be taken as evidence of
the absence of the non-selection events they would have reported.

# Privacy Considerations

SAD deliberately places on the wire information about a client's candidate
selection that would not otherwise appear there, and does so in a message
addressed to a party that, by construction, the client has decided not to
communicate with. The considerations in this section use the terminology of
{{?RFC6973}}.

## Exposure of the Five-Tuple

The exposure of transport ports in the SAD message is a deliberate design
decision, taken so that the operator of a NAT along the path can
correlate the message with its own translation state. The correlation this
enables for the NAT operator is precisely the correlation it enables for every
other observer of the message. A client behind a NAT ordinarily has its activity
aggregated with that of other clients sharing the same external address; a SAD
message restores a per-flow view of that activity to any observer positioned to
see it, including the receiving candidate itself.

A client for which this exposure is unacceptable can set the port fields to
zero, as the definition of those fields permits, at the cost of the correlation
they exist to support.

## Linkability of the Hashed DNS Answer

The hash carried in the Additional Data field is a deterministic function of the
DNS answer, and is therefore stable across every message that reports
non-selection of a candidate derived from that answer. An observer that receives
more than one such message can link those messages to one another by equality of
the field alone, without reversing the hash and without reference to the source
address of the messages. That linkage extends across time, across changes of
client address and NAT binding, and across devices, since two clients resolving
the same name will produce the same hash.

The Additional Data field is thus a linkable identifier in the sense of
{{RFC6973}}. It identifies a resolution rather than a user, and for names
resolved by very many clients the distinction is a meaningful one; for names
resolved by few, it is thin. {{not-selected-behavior}} requires that the hashed
answer be omitted where the client knows the answer was retrieved from an
off-path resolver, which addresses the related concern that SAD would otherwise
disclose to on-path parties the content of resolutions deliberately routed
around them, but it does not bear on the linkability described here, which
arises equally for answers obtained on path.

## Limits of Hashing

Hashing the Additional Data raises the cost of reading it; it does not make it
confidential. DNS answers are drawn from a space that is public, enumerable, and
for any given deployment small: an observer that wishes to learn which name a
given hash corresponds to can compute the hashes of the names it considers
plausible and compare, at a cost that does not scale with the strength of the
hash function.

The hashing in this design should accordingly be understood as limiting bulk
disclosure to incidental observers, and as providing a correlation handle to
parties that already hold the answer, which is the goal stated in
{{introduction}} — and not as withholding the answer from an observer that wants
it. This follows from the distribution of the inputs rather than from the choice
of algorithm, and registration of stronger algorithms in the Hash Algorithm
registry will not change it. A construction that did resist such an attack, such
as a keyed hash or a per-client salt, would do so by destroying the cross-party
correlation the field exists to provide.

## Sampling as a Privacy Control {#sampling-as-a-privacy-control}

The Approximate Sample Rate field described in {{approximate-sample-rate}}
allows a client to report non-selection for only a portion of its candidates.
Sampling bounds the number of linkable observations any observer can collect
from a given client over a given interval, and clients weighing the exposure
described above may reasonably treat the sample rate as a privacy parameter as
well as a measurement one.

The rate itself is visible in the message, and a rate that is unusual among the
population of clients an observer sees may serve to distinguish those clients
from the remainder. Clients selecting a sample rate for privacy reasons are
therefore better served by a value that is common among their peers than by one
chosen independently.

# IANA Considerations

This document has two actions for IANA:

- It requests the allocation of Type 44 in the ICMP Type Numbers registry for
  Slow Alternative Detection, with this document as reference on publication as
  an RFC.

- It requests the establishment the Slow Alternative Detection ICMP Code Field
  subregistry initialized with the contents of {{code}}.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to the participants in the discussions on error reporting at the HAPPY WG
meetings at IETF 123 in Madrid, IETF 124 in Montreal, and on the mailing list in
between, to which this document is an answer. Special thanks to Martin Duke for
backronyming the name of this extension, and to Martin Thomson for pointing out
interactions with ECH.

The Approximate Sample Rate and Security and Privacy Considerations sections
were written with the assistance of Claude Opus.
