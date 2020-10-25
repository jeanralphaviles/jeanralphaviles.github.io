---
layout: post
author: Jean-Ralph Aviles
title: "The Torrent Tracker HTTP Protocol"
permalink: /:categories/:year/:month/:day/:title:output_ext
description: "How Torrent clients discover peers with trackers over HTTP."
categories: torrents
tags: [torrent, tracker, http, curl]
date: 2019-11-02T15:41:00-7
# Will work once https://github.com/jekyll/minima/pull/432 is released.
modified_date: 2020-10-11T21:16:00-4
# https://github.com/jekyll/minima/pull/542
last_modified_at: 2020-10-11T21:16:00-4
---

[![alt text](/assets/pictures/Gps_Satellite.jpg "GPS Satellite")](
{% post_url 2019-11-02-tracker-http-protocol %})

# What is a Torrent Tracker

A tracker is an HTTP(s) service which helps peers of a torrent discover one
another to facilitate file sharing. Torrent
([metainfo](https://www.bittorrent.org/beps/bep_0003.html#metainfo-files))
files or [magnet links](https://www.bittorrent.org/beps/bep_0009.html) can
contain a list of trackers that the swarm (the entire network of people
connected to a single torrent) should use.

Examples of torrent trackers include <https://torrent.ubuntu.com/announce> and
<http://tracker.archlinux.org:6969/announce>.

There are a few variants of torrent trackers out there. Common ones include
[UDP trackers](https://www.bittorrent.org/beps/bep_0015.html) and
[Websocket trackers](https://github.com/webtorrent/webtorrent). There are even
["trackerless"](https://en.wikipedia.org/wiki/BitTorrent_tracker#Trackerless_torrents)
torrents which use a
[Distributed Hash Table](https://en.wikipedia.org/wiki/Distributed_hash_table)
(DHT) to find peers without a centralized tracker.

Often, torrents will be tracked by multiple different trackers and the DHT to be
redundant to failures of any one tracker.

# How does a Tracker work

When starting a download, your torrent client connects to any trackers listed
in the .torrent file or magnet link and asks for peers for that torrent. The
tracker then responds with the IP addresses and ports of other peers who have
recently connected to that same tracker. Your client will periodically ping the
tracker to ensure that the list of peers is up to date.

If a peer closes their client or stops checking in with the tracker, it will be
assumed dead and will not be handed out to other peers [^2].

![alt text](/assets/pictures/Torrent_Tracker.svg "Torrent Tracker")

# BitTorrent Spec

The BitTorrent protocol is quite simple. Instead of being standardized with
[RFCs](https://en.wikipedia.org/wiki/Request_for_Comments), as most open
protocols are, the BitTorrent protocol is defined with BEPs-- [BitTorrent
Enhancement Proposals](https://www.bittorrent.org/beps/bep_0001.html). BEPs are
modeled after the [Python Enhancement
Proposal](https://www.python.org/dev/peps/) (PEP) process[^1].

The [BEP 3](https://www.bittorrent.org/beps/bep_0003.html) specification covers
the BitTorrent protocol. This will be referenced in describing the tracker
protocol.

# Tracker HTTP Protocol

HTTP torrent trackers speak a protocol as defined in
[BEP 3](https://www.bittorrent.org/beps/bep_0003.html#trackers). Torrent clients
periodically issue HTTP GET requests to a tracker to discover peers.

GET requests have the following URL query parameters.

* info_hash
  * In the case of a .torrent file a
    [SHA1](https://en.wikipedia.org/wiki/SHA-1) hash of an encoding of the
    ["info"](https://www.bittorrent.org/beps/bep_0003.html#peer-protocol)
    section of the torrent file.
  * Listed as a parameter of a Magnet link.
* peer_id
  * Client-created 20 byte self-identifier. Used to identify peers.
* ip
  * Optional IP address (or DNS name) that the peer should be reached at.
* port
  * Port the peer is listening on.
* uploaded
  * The total amount uploaded, so far.
* downloaded
  * The total amount downloaded, so far.
* left
  * The number of bytes this peer still has to download.
* event
  * Optional key representing the state of the download: `started`, `complete`,
    or `stopped`.

This can be represented as an HTTP request with curl:

```bash
# The info_hash must be specified as URL encoded hex.
curl --get https://torrent.ubuntu.com/announce \
  --data 'info_hash=%D1%10%1A%2B%9D%20%28%11%A0%5E%8C%57%C5%57%A2%0B%F9%74%DC%8A' \
  --data-urlencode 'peer_id=01234567890123456789' \
  --data-urlencode 'port=12345' \
  --data-urlencode 'uploaded=0' \
  --data-urlencode 'downloaded=0' \
  --data-urlencode 'left=0'
```

In response, the tracker will return to you a bencoded dictionary containing two
keys: `interval` how long this client should wait before contacting the tracker
again and `peers` a list of peers to connect to. In event of an error, a single
key `failure`  will be populated instead.

Optional keys: `complete` and `incomplete` can be returned representing the
number of seeders and peers in the swarm respectively.

```bash
# Continue reading for how to decode response from the tracker to this.
{
    "complete": 2068,
    "incomplete": 177,
    "interval": 1800,
    "peers": [
        {
            "ip": "1.2.3.4",
            "peer id": "-TR2940-0890apv0oznf",
            "port": 51413
        }
    ]
}
```

Armed with a list of peers, your torrent client will connect to them and
negotiate downloads of pieces of the file. I'll write up how this works at a
later time.

This response from the tracker is
[bencoded](https://en.wikipedia.org/wiki/Bencode), an encoding method used by
BitTorrent. There aren't many bencoders out there, but this simple
[Go Playground](https://play.golang.org/p/seSAP10oaY2) can parse the response
for you if you provide a hexdump of the output.

```bash
curl --get https://torrent.ubuntu.com/announce \
  --data 'info_hash=%D1%10%1A%2B%9D%20%28%11%A0%5E%8C%57%C5%57%A2%0B%F9%74%DC%8A' \
  --data-urlencode 'peer_id=01234567890123456789' \
  --data-urlencode 'port=12345' \
  --data-urlencode 'uploaded=0' \
  --data-urlencode 'downloaded=0' \
  --data-urlencode 'left=0' | hexdump -ve '1/1 "%.2x"'
```

[^1]: <https://www.bittorrent.org/beps/bep_0000.html>
[^2]: <http://jonas.nitro.dk/bittorrent/bittorrent-rfc.html#anchor17>
