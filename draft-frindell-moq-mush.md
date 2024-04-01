---
title: "Media Using Server-Push in HTTP (MUSH)"
abbrev: "mush"
category: info

docname: draft-frindell-moq-mush-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - next generation
 - unicorn
 - april fools
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-mush"
  latest: "https://afrind.github.io/draft-frindell-moq-mush/draft-frindell-moq-mush.html"

author:
 -
    fullname: "afrind"
    organization: Meta
    email: "afrind@meta.com"

normative:

  HTTP3: RFC9114
  Priorities: RFC9218
  QuicOnStreams: I-D.draft-kazuho-quic-quic-on-streams
  MOQT: I-D.draft-ietf-moq-transport

informative:


--- abstract

This draft defines how to use {{HTTP3}} to publish and receive Tracks as defined
in MoQ Transport {{MOQT}}.  The draft does not use WebTransport, and as such
only servers can publish. Tough Cookies.

--- middle

# Introduction

HTTP/3 can totally send media.  This is one way to do it folks.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Pronunciation

MUSH rhymes with PUSH.

# Requesting Tracks

To get information about a track, a client sends a Track Info request.  To
retrieve tracks, a client sends a Fetch or a Subscribe request, or schedules a
meeting with the MoQ Chairs.

## Encoding Track Names in URLs

A Full Track Name consists of two components, a track namespace and a track
name. These are encoded in URLs with the track_namespace and track_name query
parameters.  The :path and :authority pseudo headers are set via the Connection
URL.  The parameters are URL encoded.

## Request Headers

The following headers are valid in requests.

### Priority

This header indicates the priority of this track relative to other tracks and
HTTP requests.  See {{Priorities}}.

### Other-Priority

This header is the other priority header that the working group is still
discussing, but lacks consensus. TODO. You can send both this and the Priority
header, or neither, and it means something different to every implementation.
Good luck.

### Authentication Headers

Authentication information is optionally passed by subscribers using the headers
of their choice.

## Track Info Request

Track Info is represented as an HTTP HEAD for the track.

No additional headers are allowed.

## Fetch Request

Fetch is represented as an HTTP GET for the track.  To fetch a single object,
add the group_id and object_id query parameters to the URL.  To fetch a range of
objects in a Track, omit these query parameters. The following additional
headers are allowed.

### Object-Range

The Object-Range header is a sf-dictionary with four integer fields:
start_group, start_object, end_group and end_object.  The header defines the
range of objects to be returned.  If the Object-Range is omitted, the entire
track is requested up to the end or the live-head at the time the request was
received.  Negative values are supported, and the publisher should interpret
this as a request to go back in time and turn the camera on sooner.

## Subscribe Request

Subscribe is represented as an HTTP POST to the track URL.  A client can only
have one active Subscribe request for a track at the same time.  The following
additional headers are allowed.

### Live-Start

Live-Start is an enumeration with one of the following values:

Next Object:

: The subscription begins with the next published object in the largest group
seen by the publisher or relay, or the next object if none have been published
yet.

Next Group:

: The subscription begins with the first object published in the next group with
a group ID higher than largest, or the next group if none have been published
yet.

Next Next Next Prev Prev Next Group:

: The subscription is supposed to start after the next commercial break, except
the receiver went too far forward, then too far backward before dialing it in.

## Don’t Subscribe Request

The subscriber sends this message when they don’t want to receive any track
data, but feels like they need to say something to keep the connection from
getting awkward.  It is represented as an OPTIONS request.

# Response

If the Track Info, Fetch or Subscribe request is successful, the status code is
200.  If the response is an error, the publisher will send an appropriate 400 or
500 status code.  If the publisher is feeling ornery, it can send one or more
100-continue responses before sending a real response code, because that is
totally a thing you can do in HTTP.

## Response Headers

### Object-Range

Fetch responses for a range contain an Object-Range header indicating the start
and end locations for the range.  This will be a subset of the requested range
if the requested end is beyond the end of the track or current live head.

### Live-Head

If the track is currently live and the Live Head is known, the Fetch or
Subscribe response will contain the largest group and largest object within that
group.  This header is an sf-dictionary with integer fields largest-group and
largest-object.

### Dead-Head

If the track is not currently live, the Fetch or Subscribe response contains
this header including a lyric from a Grateful Dead song.

### Delivery-Mode

This header is an enumeration of the following values:

Track:

: All objects will be delivered in a single Push stream.

Group:

: Objects in the same group will be delivered on a single Push stream.

Object:

: Each object will be delivered in its own Push stream

Byte:

: Each byte of each object will be delivered in its own Push stream

Datagram:

: Each object will be delivered as an HTTP datagram.

### Expires

This optional header specifies a time in seconds after which the publisher will
end a Subscribe request.  Subscribers SHOULD resubscribe before their
subscription expires if they wish to continue receiving the track.  This header
MUST NOT be present in a response to a Track Info or Fetch request.

Note there’s already an Expires HTTP header with a different format and meaning,
but we decided to redefine it here anyway, mostly to troll the HTTP chairs.

## Object Delivery

Objects in the response to Fetch and Subscribe are delivered via HTTP Server
Push or HTTP Datagrams according to the Delivery-Mode of the track.

That’s right, we said SERVER PUSH.  You standardized it and we’re using it.

## Object Response Headers

The HTTP Response on the Push stream MUST have :status = 200, and can carry the
following headers:

### Priority

Sets the priority of the push stream per {{Priorities}}.

### Cryority

This header is sent by the Chairs if someone brings up a priority discussion in
the middle of discussing something else.

### Delivery-Mode

The delivery mode for this track.  This MUST match the Delivery-Mode specified
in the Fetch or Subscribe response.  This mode determines the format of the
remainder of the stream.

When Delivery-Mode = Track, the remainder of the stream is zero or more of the
following fields

~~~
{
  groupID (i)
  objectID (i)
  length(i)
  payload(..)
}
~~~

When Delivery-Mode = Group, the remainder of the stream is zero or more of the
following fields

~~~
{
  objectID (i)
  length(i)
  payload(..)
}
~~~

When Delivery-Mode = Object, the remainder of the stream is

~~~
{
  objectID (i)
  payload(..)
}
~~~

When Delivery-Mode = Byte, the remainder of the stream is

~~~
{
  objectID (i)
  offset (i)
  payload(..)
}
~~~

### Group-ID

The Group-ID for all objects in the stream.  This MUST be present when Delivery-Mode is Group or Object.  This MUST NOT be present when Delivery-Mode is Track.

## Datagram Format

When Delivery-Mode is Datagram, the format of the Datagram is

~~~
{
  groupID (i)
  objectID (i)
  priority (i)
  payload(..)
}
~~~

## Response Capsules

The format of the response payload on the request stream for Fetch and Subscribe
requests is the Capsule Protocol (interleaved with PUSH_PROMSIE frames).  The
following Capsules are defined for the publisher.

### Group End

The publisher sends Group End after it sends the last object in a group.  It has
type 0xF01 and the following format

~~~
{
  groupID (i)
  largestObjectID (i)
}
~~~

### Pod

The publisher sends a Pod capsule to add an extra layer of indirection, because
frames and capsules just weren’t enough.  It has type 0xF02, and the contents
are multiplexed metadata streams formatted using {{QuicOnStreams}}.

### Object Dropped

The publisher sends Object Dropped when it resets a Data stream carrying an
Object, or skips sending an object due to buffering and/or expiry.  It has type
0xF03 and the following format

~~~
{
  groupID (i)
  objectID (i)
}
~~~

### Mic Dropped

The publisher sent something so amazing nothing further needs to be sent.  The
connection is closed immediately.  It has type 0xF03 and no fields.

### Just Dropped

The publisher sends Just Dropped to inform the subscriber there’s something new
and exciting they should subscribe to.  It has type 0xF04 and the following
format

~~~
{
  connectURL (s)
  trackNamespace (s)
  trackName (s)
  hotness (i)
}
~~~

### Reason Phrase

The publisher sends a Reason Phrase capsule to give a really clear explanation
of why something just happened.  Nothing good ever comes from it, but that
didn’t stop us.  It has a 2048 octet minimum size. It has type 0xF05 and the
following format

~~~
{
  reallyGoodReasonPhrase(s)
}
~~~

### Done

The publisher sends Done when it is done sending Data for this Track, followed
by gracefully closing the stream.  It has type 0xF06 and the following format

~~~
{
  status (i)
  reasonPhrase (s)
  contentExits (f)
  [finalGroupSent (i)]
  [finalObjectSent (i)]
}
~~~

The following status codes are defined:

|------|---------------------------|
| Code | Reason                    |
|-----:|:--------------------------|
| 0x0  | Unsubscribed              |
|------|---------------------------|
| 0x1  | Internal Error            |
|------|---------------------------|
| 0x2  | Unauthorized              |
|------|---------------------------|
| 0x3  | Track Ended               |
|------|---------------------------|
| 0x4  | Subscription Ended        |
|------|---------------------------|
| 0x5  | Going Away                |
|------|---------------------------|
| 0x6  | Expired                   |
|------|---------------------------|

## Request Capsules

The following capsules are defined for the receiver.

### Unsubscribe

The receiver sends Unsubscribe when it wants the publisher to stop sending data
for this request after a certain group.  It has type 0xF07 and the following
format

~~~
{
  endGroupID (i)
}
~~~

The publisher will stop publishing objects once it has sent the Group End for
the group with endGroupID.  It will then send a Done capsule with
status=Unsubscribed.

To unsubscribe immediately, rather than after a specific group, the subscriber
sends a STOP_SENDING frame on the request stream.

### How Do I Unsubscribe

The receiver sends a How Do I Unsubscribe capsule if they didn’t read this
specification, which explained not one, but two different ways to unsubscribe
already.  The publisher ignores this capsule and continues publishing.  It has
type 0xF08 and no fields.

## Trailers

Clients MUST support trailers in HTTP responses, but relays WONT.  If a
publisher tries sending a trailer section it will probably crash the relay or
corrupt the cache.

# Security Considerations

This protocol probably has a DoS vector in its control messages, because it
seems like we keep forgetting how to prevent ourselves from getting DoS'd in
control messages.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Lirpa Sloof assisted in the writing of this specification.
