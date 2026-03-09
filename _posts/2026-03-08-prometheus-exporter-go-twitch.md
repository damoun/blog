---
layout: single
title: "Writing a Prometheus Exporter in Go: Twitch Exporter"
categories: [ops]
tags: [prometheus, golang, twitch, monitoring]
---

I stream on Twitch occasionally, and at some point I started wanting to track metrics over time — follower counts, concurrent viewers, subscription numbers. Twitch has an API. Prometheus has a pull model. Writing a custom exporter felt like a natural project, and it turned out to be a good way to learn the Prometheus Go client library properly.

The result is [twitch-exporter](https://github.com/damoun/twitch-exporter), a small Go service that scrapes the Twitch Helix API and exposes the results as Prometheus metrics. Here's how it's structured and what I learned building it.

## The Collector Pattern

The Prometheus Go client gives you two ways to expose metrics: register gauge/counter/histogram objects and update them, or implement the `Collector` interface and compute metrics on demand at scrape time.

For an exporter that hits an external API, the Collector pattern is the right choice. You don't want background goroutines polling Twitch on a timer — you want to fetch fresh data exactly when Prometheus scrapes you. The interface is simple:

```go
type Collector interface {
    Describe(chan<- *prometheus.Desc)
    Collect(chan<- prometheus.Metric)
}
```

`Describe` sends metadata about the metrics you'll produce. `Collect` does the actual work and sends metric values. Both are called synchronously during a scrape.

## Project Structure

```
twitch-exporter/
  collector/
    channel.go       # channel info, followers, views
    stream.go        # live status, viewer count, game
    subscription.go  # subscriber count
    charity.go       # active charity campaigns
    goals.go         # creator goals
    chat_settings.go # chat configuration metrics
  main.go
```

Each file implements a separate `Collector` for a related group of metrics. They all register against the default Prometheus registry in `main.go`:

```go
prometheus.MustRegister(
    collector.NewChannelCollector(client, channelID),
    collector.NewStreamCollector(client, channelID),
    collector.NewSubscriptionCollector(client, channelID),
    collector.NewCharityCollector(client, channelID),
    collector.NewGoalsCollector(client, channelID),
    collector.NewChatSettingsCollector(client, channelID),
)
```

## A Concrete Collector

The stream collector is a good example. During a scrape it fetches the current stream data and exposes whether the channel is live, the viewer count, and the current game:

```go
type StreamCollector struct {
    client    *helix.Client
    channelID string
    live      *prometheus.Desc
    viewers   *prometheus.Desc
}

func (c *StreamCollector) Collect(ch chan<- prometheus.Metric) {
    streams, err := c.client.GetStreams(&helix.StreamsParams{
        UserIDs: []string{c.channelID},
    })
    if err != nil || len(streams.Data.Streams) == 0 {
        ch <- prometheus.MustNewConstMetric(c.live, prometheus.GaugeValue, 0)
        return
    }

    stream := streams.Data.Streams[0]
    ch <- prometheus.MustNewConstMetric(c.live, prometheus.GaugeValue, 1)
    ch <- prometheus.MustNewConstMetric(c.viewers, prometheus.GaugeValue,
        float64(stream.ViewerCount))
}
```

When the channel is offline, the collector returns 0 for `live` and omits `viewers`. Grafana handles the gaps gracefully.

## Twitch API Authentication

The Twitch Helix API requires OAuth. For a server-side application, you use Client Credentials flow — no user interaction, just a client ID and client secret:

```go
client, err := helix.NewClient(&helix.Options{
    ClientID:     os.Getenv("TWITCH_CLIENT_ID"),
    ClientSecret: os.Getenv("TWITCH_CLIENT_SECRET"),
})
if err != nil {
    log.Fatal(err)
}

token, err := client.RequestAppAccessToken([]string{})
client.SetAppAccessToken(token.Data.AccessToken)
```

Tokens expire after a few hours, so in production I refresh them before each scrape batch. The `helix` library handles the token lifecycle if you set it up correctly.

## Metric Types in Practice

Twitch metrics map cleanly to Prometheus types:

- **Gauge**: viewer count, follower count, subscriber count — values that go up and down
- **Counter**: total views on a channel — monotonically increasing
- **Gauge with label**: `twitch_channel_info{game="Minecraft", title="..."}` — metadata as labels

One thing worth noting: don't use high-cardinality values as labels. Twitch stream titles change constantly — putting them in a label would create unbounded label sets and bloat your TSDB. Store them as gauge values set to 1 with a `title` label only if the cardinality is bounded (it isn't for titles).

## Recent Additions

The exporter has grown beyond basic stream metrics. A few notable additions:

- **Charity campaigns**: exposes the current amount raised and the goal for active charity streams
- **Creator goals**: follower goals, subscription goals — the progress bars you see on stream
- **Chat settings**: whether slow mode, follower-only, or subscriber-only mode is active

These are less about operational monitoring and more about having historical data. It's interesting to see how chat settings correlate with viewer count over time, or to track a charity goal's progress in Grafana during a stream event.

## What I'd Do Differently

The main thing I'd change is adding proper error metric tracking — a gauge that counts API errors by endpoint, so I can alert when the exporter is silently failing. Right now failures just result in missing metrics, which is easy to miss.

Also, the Twitch API has rate limits (800 requests per minute), and a scrape interval of 15 seconds with multiple collectors can add up. I've stayed well under the limit, but it's worth being explicit about rate limiting rather than relying on the scrape interval being slow enough.

The full code is on [GitHub](https://github.com/damoun/twitch-exporter). PRs welcome, especially for additional metric categories I haven't gotten to yet.
