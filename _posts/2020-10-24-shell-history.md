---
layout: post
author: Jean-Ralph Aviles
title: "You Need to Timestamp Your Shell History"
permalink: shell-history.html
description: "Add timestamps to your shell history to create an accurate ledger of events."
categories: tips
tags: [history, shell, bash, timestamp, postmortem, sre]
date: 2020-10-24T21:21:00-4
---

[![alt text](/assets/pictures/Pocketwatch.jpg "Pocketwatch in hand.")](
{% post_url 2020-10-24-shell-history %})

If you work with a production environment you **need** to be recording
timestamps in your shell history. When shit hits the fan, knowing **exactly**
when you ran that deploy script or updated that database can help you resolve an
incident quicker.

After an outage, timestamped shell history can be used to stitch together
[an accurate timeline of events](https://archive.is/ZHW2i#timelinea-screenplay-of-the-incident-use-the-incident-timeline-from-the-incident-management-document-to-start-filling-in-the-postmortems-timeline-then-supplement-with-other-relevant-entries-pWsQSj2)
for an incident report or postmortem.
[Your company does write these, right](https://archive.is/a0YTc)?

It's so easy to record timestamps to your shell history, there's no excuse for
not having it. Here's how you would configure it in Bash.

```shell
# .bashrc
export HISTTIMEFORMAT='%FT%T%z: ' #  YYYY-MM-DDTHH:MM:SS±0000
```

Easy as that!

It is also useful in other situations: Read a bunch of emails and forgot how
long ago you started your load test? Want to know what you were doing at 3 pm
yesterday? Just check your shell history.

```bash
$ history
    1  2020-10-25T20:59:55-0400: git clone https://github.com/jeanralphaviles/meteogram.git
    2  2020-10-25T21:00:05-0400: cd meteogram
    3  2020-10-25T21:00:41-0400: vim internal/app/http/http.go
    4  2020-10-25T21:02:56-0400: vim app.yaml
    5  2020-10-25T21:08:32-0400: gcloud app deploy
    6  2020-10-25T21:24:04-0400: history
```
