---
title: "Certificate Transparency behind the scenes: logs, operators, cursors, and backfill"
description: "A behind-the-scenes explanation of how Certificate Transparency log lists, operators, temporal shards, tree sizes, cursors, forward freshness, and historical backfill actually fit together."
date: "2026-05-04"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - Security
  - Certificates
tags:
  - certificate transparency
  - ct logs
  - tls
  - monitoring
  - web pki
image: "./cover-imagegen.png"
image_alt: "Certificate Transparency logs shown as independent append-only ledgers with forward and backward cursor flows"
draft: true
---

Certificate Transparency is often explained from the certificate owner's point of view: a CA issues a certificate, logs it, receives Signed Certificate Timestamps, and browsers can require those SCTs before trusting the certificate.

That is the right starting point, but it leaves out the operational question a monitor has to answer:

> Where are the logs, how many are there, which ones matter today, how do we read them, and what does it mean when a dashboard says `0/2`, `1 active`, or "recent history"?

This post looks at Certificate Transparency from behind the scenes. Not as a browser, not as a certificate requester, but as a service that wants to continuously watch the public CT ecosystem.

## CT is not one database

There is no single "the CT database" to query.

Certificate Transparency is an ecosystem of independent append-only logs. Each log is a network service with its own URL, public key, state, tree size, accepted roots, and operational behavior. Logs are usually grouped by a log operator such as Google, Cloudflare, DigiCert, Sectigo, TrustAsia, Let's Encrypt, or another organization.

The CT protocol describes how a log behaves. It does not say that there must be exactly eight operators, that each operator must have two logs, or that every monitor must read the same set of logs.

That list comes from user-agent and root-program policy.

## Where a CT reader gets the map

A CT reader needs a starting map before it can query anything. The common sources are published CT log lists.

Chrome publishes CT log lists at:

- `https://www.gstatic.com/ct/log_list/v3/log_list.json`
- `https://www.gstatic.com/ct/log_list/v3/all_logs_list.json`

Apple publishes its current CT log list at:

- `https://valid.apple.com/ct/log_list/current_log_list.json`

Chrome's documentation describes two different list meanings:

- `log_list.json` contains logs that are `Qualified`, `Usable`, or `Retired` when the list was generated.
- `all_logs_list.json` contains a wider set of logs known to Chrome, including logs not in the browser compliance list.

The important part is that a reader does not invent operators. It reads a policy list, groups the entries by `operator`, and then applies its own policy scope.

As a snapshot on May 4, 2026, Chrome's `log_list.json` exposed 8 operators and 35 logs. That number is not a protocol constant. It is a daily policy-list snapshot.

You can see the shape with PowerShell:

```powershell
$list = Invoke-RestMethod 'https://www.gstatic.com/ct/log_list/v3/log_list.json'

$list.operators |
    ForEach-Object {
        [pscustomobject]@{
            Operator = $_.name
            Logs     = $_.logs.Count
            Usable   = @($_.logs | Where-Object { $_.state.usable }).Count
            ReadOnly = @($_.logs | Where-Object { $_.state.readonly }).Count
            Retired  = @($_.logs | Where-Object { $_.state.retired }).Count
        }
    } |
    Sort-Object Operator
```

## What one log entry in the list means

A log-list entry is not just a hostname. It usually tells you:

- who operates the log
- the log URL
- the log state
- the log public key or log ID material
- the Maximum Merge Delay
- the temporal interval for certificates accepted by the log
- whether the log uses the classic RFC 6962 read API or a newer static CT read path

A simplified entry looks conceptually like this:

```json
{
  "description": "DigiCert 'Sphinx2026h1'",
  "url": "https://sphinx.ct.digicert.com/2026h1/",
  "mmd": 86400,
  "state": {
    "usable": {
      "timestamp": "2025-..."
    }
  },
  "temporal_interval": {
    "start_inclusive": "2026-01-01T00:00:00Z",
    "end_exclusive": "2026-07-01T00:00:00Z"
  }
}
```

That temporal interval is one reason an operational view can show fewer logs than the raw operator count.

## Why DigiCert may be 2 logs here, not 8

Operators often run temporal shards. Instead of one giant forever log, they run logs for time windows such as `2026h1`, `2026h2`, `2027h1`, and so on.

For example, DigiCert may have several usable logs in the published list:

- `sphinx2026h1`
- `wyvern2026h1`
- `sphinx2026h2`
- `wyvern2026h2`
- `sphinx2027h1`
- `wyvern2027h1`
- future shards

If the monitoring window is May 2026, the active certificate-expiry shard is usually the `2026h1` pair. The `2026h2` and `2027` logs are real logs, but they are future temporal shards for that monitoring purpose.

So the answer to "why `0/2` and not `0/8`?" is usually:

> The provider may have 8 logs in the published list, but only 2 logs are in scope for the current lane and time window.

Those two logs are not duplicates. They are independent logs run by the same operator. A CA may submit to more than one log so that certificates can carry SCTs from multiple logs. A monitor still has to read each log separately because each log has its own append-only tree.

## Operators, logs, and tree sizes

Think of it as three layers:

| Layer | Example | What it means |
| --- | --- | --- |
| Operator | DigiCert | Organization running CT infrastructure |
| Log | `sphinx2026h1` | One append-only Merkle tree with its own URL and key |
| Entry | index `123456789` | One certificate or precertificate entry in that log |

Each log has a tree size. The tree size is simply how many entries are currently in that log's Merkle tree.

With the classic RFC 6962 API, a monitor asks:

```text
GET <log>/ct/v1/get-sth
```

The response includes the latest tree size and signed tree head. Then the monitor reads entries by index:

```text
GET <log>/ct/v1/get-entries?start=0&end=1023
```

Indexes are zero-based. If the tree size is `4,000,000,000`, the newest entry is near index `3,999,999,999`.

This is why CT monitoring naturally becomes a cursor problem.

## What a cursor is

A cursor is just "where we are" in a log.

For forward monitoring, the cursor moves upward:

```text
last seen index -> newer entries -> current tree size
```

For historical backfill, the cursor moves downward:

```text
starting index near the tip -> older entries -> target historical window
```

A reader can have more than one cursor for the same log because it may be doing different jobs:

- a forward cursor to keep up with new certificates
- a recent backfill cursor to build a chosen recent-history window
- an archive cursor to build older history, if that lane is enabled

The CT log itself does not know about "forward freshness" or "60-day backfill." Those are reader or monitor policies layered on top of CT.

## Forward freshness versus recent coverage

These two concepts are easy to mix up.

Forward freshness answers:

> Are we close to the live tip of the log?

If Cloudflare has a tree size of `4,558,702,388` and a forward cursor is also at that tip, forward freshness is current. That means new certificates are being watched as they appear.

Recent coverage answers:

> Have we walked far enough backward to cover the configured history window?

If a monitor is configured for a recent-history window, it needs to read backward until the oldest observed CT entry timestamp in each in-scope log reaches at least that window. A service may choose 7 days, 60 days, 180 days, or something else; CT does not define that number.

Those can be true at the same time:

- forward is current
- recent backfill is only 2 hours deep

That is not a contradiction. It means the monitor is caught up with new entries, but it has not yet built the historical window.

## How to read `0/2`

`0/2` should never be read as "zero work has happened" unless the dashboard says that explicitly.

It usually means:

> Out of 2 in-scope logs for this provider and lane, 0 logs have fully reached the goal.

For recent coverage, the goal might be "oldest observed timestamp is at least the configured recent-history window."

So:

```text
Cloudflare: 0/2 reached recent-history goal
Worst: 2h
Best: 26d
```

means:

- Cloudflare has 2 in-scope logs for this view.
- Neither log has completed the full configured target.
- The worst Cloudflare log has only been backfilled around 2 hours.
- The best Cloudflare log has been backfilled around 26 days.

It does not mean "0 certificates ingested." It means "0 logs fully satisfied the configured time-window objective."

## Percentages are monitor math, not CT protocol

Certificate Transparency does not return a "freshness percentage" or a "backfill percentage." The protocol gives a reader facts: tree sizes, entry indexes, timestamps, signed tree heads, and entries. Percentages are interpretation layered on top.

A good operational dashboard should make those interpretations explicit:

| Percentage | Better meaning | Not this |
| --- | --- | --- |
| Forward freshness | Whether the forward cursor is within the monitor's freshness target | Percent of all CT logs in the world |
| Recent coverage | How much of the configured history window the worst in-scope log has reached | Percent of certificates ingested |
| Active fetches | How much scheduler capacity is currently pointed at that provider or lane | Whether the provider has only that many logs |

This is why a provider can be `100%` fresh but `<1%` recent coverage. The first says the live tip is being watched. The second says the backward cursor has barely walked into the chosen history window.

## What `1 active` or `2 active` means

Active is not the same as total logs.

If a dashboard says:

```text
2 active | 2.4K/s
```

that usually means the monitor is currently fetching from two logs for that provider in that lane. It does not mean the provider only has two logs in the published ecosystem. It means the current scheduler selected two log-fetch tasks right now.

A provider can have:

- 8 logs in the published list
- 2 logs relevant to today's window
- 1 active fetch right now because the scheduler is sharing capacity
- 0 logs fully backfilled to the configured recent-history target

All four numbers can be true.

## Why log limits matter

RFC 6962 allows logs to restrict how many entries are returned by a `get-entries` request. If a client asks for too many entries, the log can return only the maximum it permits, starting at the requested index.

This means a monitor cannot assume that asking for `8192` entries always returns `8192`. A log may return fewer entries because:

- the request reached the current tree tip
- the requested range crossed an internal server limit
- the log applied rate limiting
- the log is slow or temporarily unhealthy
- the monitor requested a range that is valid but too large for that log

Some logs also return HTTP `429 Too Many Requests` or `503 Service Unavailable`. They may include `Retry-After`, but they do not always do so. A polite monitor needs per-operator cooldowns, backoff, and scheduling logic.

## RFC 6962 and Static CT logs

Most people meet CT through the RFC 6962 API:

- `get-sth`
- `get-entries`
- `get-proof-by-hash`
- `get-sth-consistency`
- `get-roots`

Newer CT infrastructure can expose a Static CT API read path. Static CT still serves the same purpose, but the monitor reads checkpoints and immutable tiles instead of calling dynamic `get-entries` ranges for everything. The Static CT API was designed to be more cacheable and efficient for large-scale monitoring.

For a monitor, this matters because "read this log" can mean two different implementations:

| API style | Monitor reads |
| --- | --- |
| RFC 6962 | `get-sth`, `get-entries`, proofs |
| Static CT API | checkpoints, tiles, static entry data |

The published log-list metadata tells the monitor which shape to expect.

## What about older history?

Certificate Transparency logs may contain far more data than a dashboard's recent-history window. A 60-day window is not a CT limit. It is one possible monitoring policy.

A service may choose to run:

- a live forward lane
- a recent backfill lane, such as 60 days
- an archive lane, such as multiple years

If a dashboard only shows one recent-history window, that usually means the current operational view is focused on the recent lane. Older history may be disabled, deferred, or shown elsewhere. CT itself does not remove older entries from the log just because a monitor chooses a shorter operational window.

This distinction is important:

```text
CT log retention: defined by the public log and its lifecycle
Monitor retention: defined by your service configuration
Dashboard window: defined by what the UI chooses to expose
```

## A practical way to read a CT monitor dashboard

When looking at a CT monitor dashboard, ask these questions in order:

1. Which log-list source is being used?
2. Which operators are in the merged list?
3. Which logs are in scope after state and temporal filtering?
4. Is the forward cursor near the current tree size?
5. Has the backfill cursor reached the configured time window?
6. How many log fetches are active right now?
7. Are any logs rate-limiting or returning partial batches?

That turns confusing labels into a workflow.

For example:

```text
Provider: Cloudflare
Raw list: 2 Cloudflare logs
In scope today: 2 logs
Forward freshness: current
Recent coverage: worst 2h, best 26d of configured window
Recent goal: 0/2 logs have fully reached the target
Live activity: 1 recent fetch active right now
```

That is no longer "Cloudflare is doing nothing." It is:

> Cloudflare live monitoring is current, but the historical backfill is incomplete, and the worst in-scope log still needs most of the configured recent-history window.

## Sources

- [Certificate Transparency: how CT works](https://certificate.transparency.dev/howctworks/)
- [Chrome's CT log lists](https://googlechrome.github.io/CertificateTransparency/log_lists.html)
- [Chrome Certificate Transparency log lifecycle](https://googlechrome.github.io/CertificateTransparency/log_states.html)
- [Chrome Certificate Transparency log policy](https://googlechrome.github.io/CertificateTransparency/log_policy.html)
- [RFC 6962: Certificate Transparency](https://www.rfc-editor.org/rfc/rfc6962)
- [Static Certificate Transparency API](https://c2sp.org/static-ct-api)
- [Apple Certificate Transparency policy](https://support.apple.com/en-ca/103214)
