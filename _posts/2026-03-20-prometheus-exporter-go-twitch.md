---
layout: single
title: "twitch_exporter: Monitoring My Twitch Channel with Prometheus"
categories: [dev]
tags: [prometheus, golang, twitch, monitoring]
---

I started streaming on [twitch.tv/dam0un](https://twitch.tv/dam0un) in July 2017 because I wanted to share the games I loved with people who enjoyed watching them. For five years I went live four times a week, playing Nintendo titles, co-op survival games, and the occasional RPG. Getting raided by a channel with 500+ viewers was always a rush: the chat would explode, new followers would pour in, and then things would settle back to the usual 10 viewers. Over those years the channel grew to 2.2K followers across 96 different games and 747 active streaming days. In October 2022 I stopped streaming when my daughter was born, but the channel and its data are still there.

The Twitch dashboard shows you numbers, but only short-term ones. I wanted to see long-term trends: how follower growth changed after a raid, which games kept viewers around the longest, whether certain time slots worked better than others. That data existed in the Twitch Helix API, but there was no easy way to store and graph it over months or years. As an SRE who works with the Prometheus stack daily, the answer felt obvious: write an exporter, scrape the API, push everything into Grafana, and let the graphs tell the story.

## What is twitch_exporter

[twitch_exporter](https://github.com/damoun/twitch_exporter) is a Go service that scrapes the Twitch Helix API and exposes metrics for Prometheus. Thanks to [Jack Stupple (@surdaft)](https://github.com/surdaft), it now uses the Collector pattern: instead of updating metrics on a timer, each collector fetches fresh data exactly when Prometheus scrapes the `/metrics` endpoint.

```go
type Collector interface {
    Describe(chan<- *prometheus.Desc)
    Collect(chan<- prometheus.Metric)
}
```

Each collector implements this interface for a specific group of Twitch data. The exporter currently has 14 collectors covering different aspects of a channel. When Prometheus hits `/metrics`, every enabled collector fires off its API calls, transforms the responses into typed metrics, and sends them down the channel. No background goroutines, no stale data.

## What it tracks

The two metrics I cared about most were **follower growth** and **game labels**. After a raid I could watch the follower count climb in Grafana, then see exactly when the bump plateaued. Game labels on stream metrics let me compare: did Mario Kart sessions hold viewers longer than Valheim? Over months, the patterns became clear in ways the Twitch dashboard never showed.

Beyond those, the exporter covers subscriptions, bits, charity campaigns, creator goals, chat settings, and more. The full list is in the [repo](https://github.com/damoun/twitch_exporter).

## v1.1.0: a big step forward

Two weeks ago I released [v1.1.0](https://github.com/damoun/twitch_exporter/releases), the biggest update since the initial release. The headline feature is EventSub integration for chat message tracking. Polling the API does not give you chat messages; EventSub is the only way to get them, so this opened up a whole category of metrics that was not possible before.

The release also added user-scoped token support, a Helm chart for Kubernetes, and distroless Docker images (amd64 and arm64). You can now disable individual collectors at runtime to only scrape what you need.

## Contributors

I was genuinely surprised when people started contributing to a niche Twitch/Prometheus project. [Jack Stupple (@surdaft)](https://github.com/surdaft) contributed the Collector pattern refactor, the Helm chart, the disable-collectors feature, and the EventSub chatters integration. [Carlos Colaco (@coolapso)](https://github.com/coolapso) contributed improvements as well. Their PRs raised the quality bar of the project and shaped a lot of what went into v1.1.0.

## Try it out

If you stream on Twitch and run Prometheus, give [twitch_exporter](https://github.com/damoun/twitch_exporter) a try. I don't stream anymore, but I still enjoy maintaining the project and building tools for the streaming community. Feedback, bug reports, and PRs are all welcome -- open an issue or drop a comment below.
