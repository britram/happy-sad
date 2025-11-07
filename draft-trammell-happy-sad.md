---

title: "Slow Alternate Detection for Happy Eyeballs"
category: info

docname: draft-trammell-happy-sad
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: WIT
workgroup: HAPPY Working Group
keyword:
 - icmp
 - happy eyeballs
 - debugging
 - measurement
venue:
  group: HAPPY
  type: Working Group
  mail: happy@example.com
  github: trammell/happy-sad

author:
 -
    fullname: Brian Trammell
    organization: Google Switzerland GmbH
    email: ietf@trammell.ch

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
and is depicted in {{#fig-sad-message}}

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
{{#not-selected-behavior}} for the use of this message.

When present, the Additional Data field of a Not Selected contains the DNS
Message {{!RFC1035}} associated with the answer that 

## Hash Algorithm (HAlg)

The Hash Algorithm field determines both the length of the Additional Data field
and the hash algorithm used to hash it. The following hash algorithms are
available:

| Value | Length | Algorithm                                                |
|-------|--------|----------------------------------------------------------|
|     0 |      0 | Additional Data Omitted                                  |
|     1 |    256 | SHA256 {{!RFC6234}}                                      |
|  2-15 |  undef | Reserved for Future Use                                  |

## Approximate Sample Rate

TODO: allows the sender to expose a sample rate, if applied. Design an encoding
for common useful sample rates.

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

If the client knows that its DNS Answer message was retrieved from an off-path
resolver (e.g., via DoH {{?RFC8484}}), it MUST NOT include the hashed DNS Answer
in the Not Selected message.

# Security Considerations

TODO analyze various attacks against this approach.

# IANA Considerations

This document has two actions for IANA:

- It requests the allocation of Type 44 in the ICMP Type Numbers registry for
  Slow Alternative Detection, with this document as reference on publication as
  an RFC. 

- It requests the establishment the Slow Alternative Detection ICMP Code Field
  subregistry initialized with the contents of {{#code}}.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to the participants in the discussions on error reporting at the HAPPY WG
meetings at IETF 123 in Madrid, IETF 124 in Montreal, and on the mailing list in
between, to which this document is an answer. Special thanks to Martin Duke for
backronyming the name of this extension.
