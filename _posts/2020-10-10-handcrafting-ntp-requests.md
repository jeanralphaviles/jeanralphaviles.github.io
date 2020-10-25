---
layout: post
author: Jean-Ralph Aviles
title: "Handcrafting NTP Requests"
permalink: /:categories/:year/:month/:day/:title:output_ext
description: "Learn the NTP and SNTP protocol through example."
categories: ntp
tags: [ntp, sntp, time, protocol, rfc5905, rfc4330]
date: 2020-10-10T12:00:00-4
# Will work once https://github.com/jekyll/minima/pull/432 is released.
modified_date: 2020-10-11T23:58:00-4
# https://github.com/jekyll/minima/pull/542
last_modified_at: 2020-10-11T23:58:00-4
---

[![alt text](/assets/pictures/Ntp_Clock.jpg "Clock")](https://www.flickr.com/photos/matt_gibson/3281131319)

## Introduction

This post introduces NTP, and its younger sibling SNTP, and teaches you about
them with a hands-on example. This is way more detail than most people need to
know about NTP, but if you're anything like me you take interest in learning the
smaller details.

In this post we will:

1. Learn about NTP and SNTP.
1. Craft our own SNTP request and send it to a SNTP server on the command line.
1. Decipher the SNTP response to get the current time.

## NTP Overview

The Network Time Protocol (NTP) is widely used to accurately synchronize clocks
across the Internet. It is one of the oldest Internet protocols still in use,
being invented in 1985. It is ubiquitous, and nearly every Internet connected
device will have an NTP client.

Currently described by [RFC 5905](https://tools.ietf.org/html/rfc5905), NTP
consists of intricate algorithms for synchronizing time over an unreliable, and
potentially hostile, network.

The [Simple Network Time Protocol (SNTP)](https://tools.ietf.org/html/rfc4330)
is a subset of NTP intended for applications with less stringent accuracy
expectations. It foregoes some of NTP's advanced algorithms to be much simpler
to implement. It is preferred in embedded applications where CPU resources are
at a premium.

While NTP clients may be accurate within a few tenths of milliseconds over the
public Internet, SNTP clients can see inaccuracies of several fractions of
seconds.

In the rest of this post, we will cover SNTP as it is much simpler to work with.

## NTP/SNTP Message Format

Figure 1 describes the NTP/SNTP message format. These messages are passed back
and forth between NTP clients and servers as UDP packets. In client messages,
most of these fields can be ignored.

```text
                   Figure 1 - NTP Message Format
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9  0  1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|LI | VN  |Mode |    Stratum    |     Poll      |   Precision    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Root  Delay                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Root  Dispersion                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Reference Identifier                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                                |
|                    Reference Timestamp (64)                    |
|                                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                                |
|                    Originate Timestamp (64)                    |
|                                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                                |
|                     Receive Timestamp (64)                     |
|                                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                                |
|                     Transmit Timestamp (64)                    |
|                                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Leap Indicator (LI)

Two bits used as a warning of an impending leap second.

### Version Number (VN)

Three bits indicating the NTP version number, currently 4.

### Mode

Three bits indicating the protocol mode of a message. SNTP only deals with three
modes: client, server, and broadcast.

| Mode | Meaning   |
|------|-----------|
| 3    | client    |
| 4    | server    |
| 5    | broadcast |

### Stratum

Eight bits representing the "stratum" of a server's response.

Stratum represents a distance from a high-precision, high-accuracy clock.

An accurate timekeeping device such as an atomic clock or a GPS receiver is
given a stratum value of 0.

* An NTP server synchronized with a stratum 0 device has a stratum value of 1.
* An NTP server synchronized with a stratum 1 server has a stratum value of 2.
* ...

Generally you want to synchronize with an NTP server with a lower stratum value
as it will be more accurate.

![alt text](/assets/pictures/Ntp_Stratum.svg "NTP Stratum")

### Poll Interval (Poll)

Eight bit exponent of two representing the maximum interval between successive
NTP messages in seconds.

### Precision

Eight bit exponent of two representing the precision of a response.

### Root Delay

32 bits indicating the propagation delay between a server and its primary
reference source.

### Root Dispersion

32 bits indicating the maximum error in a response.

### Reference Identifier

32 bits identifying the primary reference source.

For stratum 1 servers, these identifiers are described in
[RFC 4330](https://tools.ietf.org/html/rfc4330#section-4).

For stratum 2+ responses, this value is the IPv4 address of the server's
synchronization source.

### Reference Timestamp

The time the server's system clock was last set or corrected.

### Originate Timestamp

The time at which the request departed the client for the server.

### Receive Timestamp

The time at which the request arrived at the server.

### Transmit Timestamp

The time at which the reply departed the server.

## Handcrafting a SNTP Request

Of all the NTP message fields described above, only two are required in a client
request: `Version Number` and `Mode`. All other fields can be ignored.

Clients can optionally set the `Transmit Timestamp` in their request to
calculate the propagation delay between server and client. To simplify our
example, we will omit it.

Our 48 byte client request will be almost entirely empty except for the `Version
Number` (4) and `Mode` (3).

Filling in the first byte of the NTP message gets us:

```text
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|0 0|1 0 0|0 1 1| == 0x23
+-+-+-+-+-+-+-+-+
```

As a hexdump, the full request packet would be:

```text
00000000: 2300 0000 0000 0000 0000 0000 0000 0000  #...............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

Exciting stuff isn't it? Save this file as `request.dump`, or
[download it here](https://raw.githubusercontent.com/jeanralphaviles/jeanralphaviles.github.io/master/assets/files/request.dump).

## Sending our Request

We will send our request to an NTP server at [pool.ntp.org](http://pool.ntp.org)
which listens on udp port 123. We will utilize
[netcat](http://netcat.sourceforge.net) and `xxd` to send the raw bytes of our
request.

```bash
$ xxd -r request.dump | nc -uw1 pool.ntp.org 123 | xxd
00000000: 2401 00e9 0000 0000 0000 0048 5050 5300  $..........HPPS.
00000010: e32c 49c6 e79d 9ea3 0000 0000 0000 0000  .,I.............
00000020: e32c 49ce abba bde0 e32c 49ce abbc b6c9  .,I......,I.....
```

And we got back a response!

## Decoding the Response

Now that we have a response, let's decode it. We'll use `xxd` to output our
NTP response in a format that is easier to work with.

```bash
$ xxd -r request.dump | nc -uw1 pool.ntp.org 123 | xxd -c 4
00000000: 2401 00e9  $...
00000004: 0000 0000  ....
00000008: 0000 0048  ...H
0000000c: 5050 5300  PPS.
00000010: e32c 49c6  .,I.
00000014: e79d 9ea3  ....
00000018: 0000 0000  ....
0000001c: 0000 0000  ....
00000020: e32c 49ce  .,I.
00000024: abba bde0  ....
00000028: e32c 49ce  .,I.
0000002c: abbc b6c9  ....
```

Here each line corresponds to a single line in Fig 1.

In this example we will extract the `Transmit Timestamp` to get the current
time. From Fig. 1 we know that the `Transmit Timestamp` corresponds to the last
two lines (8 bytes) of output.

```text
00000028: e32c 49ce  .,I.
0000002c: abbc b6c9  ....
```

SNTP uses the standard NTP timestamp format described in
[RFC 1305](https://tools.ietf.org/html/rfc1305). Timestamps are represented as a
64-bit unsigned fixed-point number, in seconds relative to `0h` on `1 January
1900`. The integer part is in the first 32 bits, and the fraction part in the
last 32 bits. These timestamps can represent instants in time over a period of
136 years with an accuracy of 232 picoseconds.

```text
                 Figure 2 - NTP Timestamp Format
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Seconds                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Seconds Fraction (0-padded)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Knowing this, the first 4 bytes `0xe32c49ce` represents the number of seconds
elapsed since `1 January 1900`. We won't worry about fractions of a second.

With the Unix `date` command we can convert this into a human readable string.
To do this we must first subtract the number of seconds between `1 January 1900`
and `1 January 1970` as the `date` command expects seconds since the
[UNIX Epoch](https://en.wikipedia.org/wiki/Unix_time). I happen to know that
there are `2208988800` seconds between these two dates.

With simple arithmetic we can decode the timestamp.

```bash
$ date -d @$(( 0xe32c49ce - 2208988800 )) # GNU/Linux
$ date -r $(( 0xe32c49ce - 2208988800 ))  # BSD/OSX
Sat Oct 10 10:55:10 EDT 2020
```

Voilà!

## Conclusion

I hope you enjoyed this post as much as I enjoyed writing it. If you notice that
I got any of the details wrong please feel free to let me know.

Thank you!

## References

* [RFC 4330](https://tools.ietf.org/html/rfc4330)
* [RFC 5905](https://tools.ietf.org/html/rfc5905)
* [Wikipedia](https://en.wikipedia.org/wiki/Network_Time_Protocol)
