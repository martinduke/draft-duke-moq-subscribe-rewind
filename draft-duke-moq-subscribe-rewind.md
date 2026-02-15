---
title: "The Rewind Subscription Filter"
abbrev: "moq-subscribe-rewind"
category: std

docname: draft-duke-moq-subscribe-rewind-latest
submissiontype: IETF
ipr: trust200902
date: {DATE}
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - subscribe
venue:
  group: Media Over Quic
  type: Working Group
  mail: moq@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/moq/ 
  github: martinduke/draft-duke-moq-subscribe-rewind
  latest: https://martinduke.github.io/draft-duke-moq-subscribe-rewind/draft-duke-moq-subscribe-rewind.html

author:
 -
    fullname: Martin Duke
    organization: Google
    email: martin.h.duke@gmail.com

stand_alone: true

normative:
  MOQT: I-D.ietf-moq-transport

informative:

--- abstract

This document proposes a Media Over Quic Transport (MOQT) extension that enables
a new Subscription Filter, so that a subscriber can request that a finite number
of past groups be delivered with SUBSCRIBE semantics (multiple streams,
potentially incomplete) rather than FETCH semantics (single stream, complete,
head-of-line-blocking). Service of this request is best-effort by the publisher.

--- middle

# Introduction

In MOQT {{MOQT}}, tracks are delivered via atomic Objects that are organized
into Groups, which serve as join points, and Subgroups, that imply dependency
between Objects and serve as the units to be fed into QUIC or Webtransport
streams.

Subscribers can send SUBSCRIBE messages to receive messages that arrive at, or
are created by, the publisher in the future. Such messages are delivered either
one stream per Subgroup, or in QUIC Datagrams, as dictated by the original
publisher. The stream mapping allows Objects that are no longer of use to the
subscriber to not be sent or retransmitted without blocking later Objects.

Subscribers can also send FETCH messages to retrieve Objects from the past. The
requested Object range is delivered on a single stream and cannot omit Objects
that exist at the original pulbsher. If the entirely of the Object range is not
in cache, a relay will have to issue its own FETCH upstream to satisfy the
subscriber.

Because the subscriber may not know the live edge at request time, a variant of
FETCH known as "Joining FETCH" instructs the publisher to use the current live
edge as the end of the Object range. A "Relative Joining FETCH" defines the
start of the Object range relative to the live edge. For instance, a Relative
Joining FETCH might request two Groups prior to the live edge, which would
deliver the two latest complete Groups as well as all Objects in the current
Group before the live edge. Joining FETCH uses the same delivery semantics as
other FETCH: all Objects are delivered in order on a single stream.

In some use cases, this behavior is not optimal. The subscriber might not need
the delivery guarantees associated with FETCH if Objects will arrive too late
to be useful. Furthermore, if some of the FETCHed Objects are available in
cache, they might have to wait for other, blocking Objects to be delivered from
upstream.

This document describes the Subscribe Rewind extension, which specifies a new
Subscription Filter type "Rewind", allowing the subscriber to request SUBSCRIBE
semantics for some groups before the live edge. The publisher only honors the
request if it is able to do so from cache.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Overview

Each endpoint sends the MAX_REWIND option in its SETUP message. The MAX_REWIND
option contains an integer that indicates the maximum number of groups that
the peer migth request in a Rewind subscription filter. If zero, the peer may
only request the current group. If absent, the peer MUST NOT send the Rewind
subscription filter. This option is half-duplex; if an endpoint does not send
the option, but receives it, it MAY use the Rewind Subscription Filter.

An endpoint might populate the MAX_REWIND option by reporting how many Groups
it habitually stores in cache to answer FETCH or service subscribers that are
behind on delivery. Other heuristics are also possible.

A subscriber sends a SUBSCRIBE message and can include a Rewind Subscription
Filter instead of some other Subscription filter type. A value of zero
indicates it would like to receive the entire current group; a larger value
indicates it would also like to receive the most recent complete groups as well.

If the subscriber wants the Groups even if SUBSCRIBE semantics are not
available, it MAY also send a Joining FETCH message. The object range MAY be
larger or smaller than specified in the Rewind filter.

Upon receipt of a Rewind filter, the publisher MAY treat it as a Largest Object
filter. It will typically do so if the track is not in cache. If it does not
do so, it sends a "StartGroup" parameter in the SUBSCRIBE_OK. StartGroup
is an integer that indicates the number of Groups before the LargestObject
parameter that will be served via SUBSCRIBE.

Groups included in the StartGroup range will be delivered using SUBSCRIBE
semantics: datagrams or Subgroup streams, subject to the delivery timeout
and group order specified in the SUBSCRIBE negotiation. Groups serviced via
the Rewind filter are not delivered by any Joining FETCH associated with this
SUBSCRIBE, though can be delivered by Standalone FETCH messages. In some cases,
this means the Joining FETCH delivers an empty range.

# Publisher restrictions

The publisher MUST NOT include a Group in a range defined by the StartGroup
unless:

* There are objects in cache for the group.
* The DELIVERY_TIMEOUT parameters for the SUBSCRIBE indicate the Group can
still be sent.

If the publisher is a Relay, it must also meet one of the following conditions
for the objects populating the cache from that Group:

* The cache was fully populated by FETCH message(s) that cover the entire
Object ID range of the Group;

* The cache was fully populated by an active upstream SUBSCRIBE with a Largest
Object before the Group;

* The cache was populated by FETCH message(s) that continuously covers a range
starting at Object ID 0, and an active upstream SUBSCRIBE has Largest Object
equal to or less than the end of that range.

The publisher is not required to verify that it has all objects in the Group to
include it a StartGroup range. In particular, Groups delivered via SUBSCRIBE
might be missing objects and are still eligible for Rewind.

The publisher MAY choose to report fewer groups than what meet these conditions.
It might do so because the volume of data implied would consume too many
resources, because it knows the current group is about to end, or due to any
other policy.

As with any other SUBSCRIBE, if a publisher receives two streams for the same
Subgroup from upstream, and cannot account for all object IDs between the end
of one and the beginning of another, it MUST NOT deliver them on the same
stream. It MAY simply omit the stream with higher object IDs.

The publisher MUST NOT send StartGroup > 0 if it knows it is servicing a track
with Group IDs that are not strictly increasing.

# Options and Parameters

## Setup Option MAX_REWIND

In addition to the Setup Options in Sec 9.3.1 of {{MOQT}}, the Setup Option
MAX_REWIND (0x16) contains an integer that indicates the largest value that can
be used in a Rewind Subscription Filter. If it is missing, the peer MUST NOT
send a Rewind Subscription Filter.

## Subscription Filter Rewind

In addition to the Subscription Filter Types in Sec 5.1.2. of {{MOQT}}, add
filter type Rewind (0x16). The format is as follows:

~~~
Subscription Filter {
  Filter Type (vi64) = 0x16,
  Start Group (vi64),
}
~~~

A StartGroup of zero means that the subscriber requests SUBSCRIBE semantics
from the beginning of the current group. A larger integer value includes that
many past Groups in addition to the current Group.

The StartGroup field MUST NOT exceed the value in the peer's MAX_REWIND Setup
Option and the filter type MUST NOT be sent if the Option was absent. In either,
case, the publisher should terminate the session with error PROTOCOL_VIOLATION.

## START_GROUP Message Parameter

In addition to the MessageParameters in Sec 9.2 of {{MOQT}}, add StartGroup
(0x16).

It represents the number of groups before the LargestObject that will be
delivered via SUBSCRIBE semantics.

If the parameter is sent in response to a Subscription Filter other than
Rewind, has a value greater than the Group ID of Largest Location, or exceeeds
the Start Group field of the filter, the subscriber MUST close the session
with error PROTOCOL_VIOLATION.

The publisher MUST truncate the end of any Joining FETCH related to this
SUBSCRIBE to end before Object zero of the Group encoded by START_GROUP. This
might result in empty object ranges. The Subscriber MUST close the session
with error PROTOCOL_VIOLATION if the ranges overlap.

# Security Considerations

To the extent this reduces head-of-line-blocking and replaces upstream FETCHes
to satisfy Joining FETCH at the relay, it can reduce resource consumption at
publishers.

However, SUBSCRIBE semantics consume more QUIC or Webtransport streams. Past
Groups might contain a lot of data, and FETCH delivery is contained on a
single stream to simplify the flow control of this data. Publishers, who are
aware of the content of their cache, SHOULD limit the range encoded by the
START_GROUP parameter when is likely to overwhelm the channel or the subscriber.

# IANA Considerations

Please add 0x16 (START_GROUP) to the Message Parameters registry.

There is no Setup Option registry, but if one arises, please add 0x16
(MAX_REWIND) to it.

--- back

# Acknowledgments
{:numbered="false"}

Ian Swett designed many components of this proposal. Numerous members of the
MOQ Working Group provided comments that refined our thinking.
