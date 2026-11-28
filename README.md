# BetterPOV provider hints

This repository is published automatically and holds exactly three files: `v1/providers.json`, `.nojekyll` and this README. Every update replaces the whole repository with a single new commit, so it has no history. Please don't open pull requests: they will be overwritten.

**File:** <https://antoniegeerts.github.io/betterpov-hints/v1/providers.json>

## What it is

BetterPOV (a Kodi video add-on) searches several source providers at once. Each install keeps its own small scorecard on the device. This file adds network-wide averages, so a new install starts with sensible defaults.

Installs that turn on **Settings › Providers › Help improve search speed** (it is off by default) send one anonymous set of totals per day:
- for each provider: the number of searches, timeouts, errors, empty answers, results and cached answers;
- how often a search ended in a play, and how often the played stream came from that provider;
- typical and slow response times;
- the date and the add-on version.

The upload contains no identifier, titles, searches, links or accounts. A scheduled job combines the last 3 days into this file twice a day. Raw uploads are never published, and they expire after 4 days. Privacy policy: <https://betterpov.vercel.app/privacy>

How the job combines them:
- each upload is one vote, however many searches it claims;
- a provider is listed only when at least 5 uploads, each with at least 2 searches, mention it;
- response times are medians, and rates are 10% trimmed means;
- a guard keeps the previous file when upload volume jumps or many providers change state at once.

## What the add-on does with it

Every install downloads this file at most once a day. It validates the file strictly: unknown fields and providers it does not have are ignored, and the file is ignored if it is older than 7 days. A hint can only:
- reorder the providers the user already has enabled;
- let a provider that is failing for most users sit out for about 2 hours (it is still retried, and one local success overrides the hint);
- shorten a provider's timeout, within fixed limits and never beyond the user's own setting.

It can never add providers, links, code or settings. Once a device has 10 or more searches of its own for a provider, its own figures replace the hint for that provider.

## Schema (`v: 1`)

```json
{"v":1,"generated":"2026-11-26T14:02:11Z","through":"2026-11-25","window_days":3,"installs":412,
 "providers":{"torrentio":{"n":380,"lat_p50":1800,"lat_p90":4200,"timeout_rate":0.03,"error_rate":0.01,
   "play_rate":0.42,"down":false,"weight":0.82,"timeout_s":8}},
 "meta":{"installs_by_day":{"2026-11-25":412}}}
```

**Top-level keys.** No other keys are allowed.

| Key | Type | Meaning |
|---|---|---|
| `v` | 1 | Schema version |
| `generated` | `YYYY-MM-DDTHH:MM:SSZ` | When the file was built (UTC) |
| `through` | `YYYY-MM-DD` | Last full UTC day included |
| `window_days` | integer | Days combined (3) |
| `installs` | integer | Uploads received for `through` |
| `providers` | object | At most 60 entries, keyed by provider module name (`^[a-z0-9]{2,24}$`). Empty when the window has fewer than 20 uploads |
| `meta` | object, optional | `installs_by_day`: uploads per day, used by the job's volume guard. The add-on ignores it |

**Provider fields.** Each entry has exactly these 9 fields.

| Field | Type and range | Meaning |
|---|---|---|
| `n` | integer, 5 to 1 000 000 | Uploads (votes) behind this entry |
| `lat_p50` | integer ms, 0 to 60 000 | Median of the uploads' typical response time (0 = none reported) |
| `lat_p90` | integer ms, 0 to 60 000, at least `lat_p50` | Median of the uploads' slow response time |
| `timeout_rate` | number, 0 to 1 | Trimmed mean share of searches that timed out |
| `error_rate` | number, 0 to 1 | Trimmed mean share of searches that failed |
| `play_rate` | number, 0 to 1 | Trimmed mean share of plays that came from this provider (0.3 when unknown) |
| `down` | boolean | At least 10 uploads for `through`, and at least 80% of them saw this provider fail at least 80% of the time |
| `weight` | number, 0 to 1 | `0.5·play_rate + 0.3·(1 − timeout_rate − error_rate) + 0.2·clamp((10000 − lat_p50)/9000, 0, 1)` (speed counts as 0.5 when no latency was reported), rounded to 2 decimals |
| `timeout_s` | integer, 5 to 30 | `ceil(1.5·lat_p90/1000 + 1)`, clamped (30 when no latency was reported) |

The file is at most 8 KB.
