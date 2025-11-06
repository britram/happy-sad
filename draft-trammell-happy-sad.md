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

TODO improve: one of the issues with happy eyeballs is that it masks
configuration and deployment failures of the very protocols in the Internet that
it is designed to accelerate the deployment of. Debugging and measuring these
failures is difficult. This document specifies Slow Alternate Detection (SAD)
for Happy Eyeballs, a method for exposing failure information to servers and 
on-path devices in order to aid debugging and measurement of Happy Eyeballs.

--- middle

# Introduction

TODO: frontmatter goes here. 

TODO: design principle: never reflect data, but send hashes of received data
s.t. DNS information can only be recognized by an entity that already has the
full data, supporting correlation for detailed debugging while supporting only
aggregated measurement for 


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Message Format (ICMP)

The message format for SAD is identical for both ICMPv4 and ICMPv6,
and is depicted in (TODO figurize).

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Type = 44   |     Code        |     HAlg    |   reserved    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   |                     Additional Data                           |
   |                     (code-dependent)                          |
   ...                                                           ...

## Initial Code Subregistry

The ICMP Code for a SAD message takes one of the following values:

| Code | Description                      | Additional Data       |
|------|----------------------------------|-----------------------|
|    0 | Not Selected                     | DNS Answer hash       |
|    1 | Alternate A Resolution Failed    | DNS Answer hash       |
|    2 | Alternate AAAA Resolution Failed | DNS Answer hash       |
|    3 | Alternate Transport Timeout      | DNS Answer hash       |
|    4 | Alternate ICMP Error             | ICMP datagram hash    |

### Not Selected

TODO: on-path message, send to expose that a path was not selected. Hash the DNS
answer. Determine timing.

### Alternate (A/AAAA) Resolution Failed

TODO: off-path message: send along the selected path when the resolution of a
not selected path failed. Two codepoints: one for each failed AF. Hash the error
answer.

### Alternate Transport Timeout

TODO: off-path message, send along the selected path when the non-selected path
failed due to connection timeout. Hash the DNS answer.

### Alternate ICMP Error

TODO: off-path message, send along the selected path to notify that the
non-selected path failed with an ICMP error. Hash the ICMP datagram of the
error.

## Additional Data

The Additional Data field, if present, MUST be hashed with a hash algorithm
specified by the 

| Value | Length | Algorithm                                                |
|-------|--------|----------------------------------------------------------|
|     0 |      0 | Additional Data Omitted                                  |
|     1 |    256 | SHA256 (TODO ref)                                        |
| 2-255 |  undef | Reserved for Future Use                                  |


# Protocol Behaviors

## Clients

TODO: optional. can (should?) also use sampling. 

## Servers 

TODO: should log if present. 

## On-Path Devices

TODO: may log, without access to answers/icmp messages this still allows for
aggregate analysis.

# Security Considerations

TODO security is a good idea. consider the shape of the Alternate codepoints in
particular, because they mix paths they seem like they will be hard to do
safely.

# IANA Considerations

This document has two actions for IANA:

- It requests the allocation of Type 44 in the ICMP Type Numbers registry for
  Slow Alternative Detection, with this document as reference on publication as
  an RFC. 

- It requests the establishment the Slow Alternative Detection ICMP Code Field
  subregistry initialized with the contents of {{#initial-code-subregistry}}.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to the participants in the discussions on deployment at the HAPPY WG
meeting at IETF 124 in Montreal for the ideas that led to this draft, to Tommy
Pauly for the suggestion to use ICMP, and to Martin Duke for the name of this
extension.
