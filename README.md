# Kouro - Flight Search & Aggregation

Go flight search and aggregation engine. Aggregates flights from four
mock airline providers, normalizes heterogeneous response shapes into a single
schema, and exposes the result over HTTP.

## Requirements

- Go 1.22+
- [jq](https://jqlang.github.io/jq/) (for pretty-printing API responses, optional)

```bash
scoop install jq        # Windows (Scoop)
brew install jq         # macOS
sudo apt install jq     # Ubuntu/Debian
```

## Run

```bash
cd BookCabin
go mod tidy
go run ./cmd/server
```

On startup the server automatically launches four airline mock servers on
random local ports and logs their addresses:

```
airline mock: Garuda Indonesia      http://127.0.0.1:54321
airline mock: Lion Air              http://127.0.0.1:54322
airline mock: Batik Air             http://127.0.0.1:54323
airline mock: AirAsia               http://127.0.0.1:54324
kouro listening on :8080
```

The main API is always on `:8080`. Airline servers pick free ports automatically
so there are no port conflicts.

Custom address or cache TTL:

```bash
go run ./cmd/server -addr :9000 -cache-ttl 1m
```

**Important:** run from inside the `BookCabin/` directory. The server reads
mock data from `test/testdata/` relative to the working directory.

## API

The backend exposes exactly two endpoints — nothing else.

| Method | Path | Description |
|---|---|---|
| `GET` | `/healthz` | Health check — returns `{"status":"ok"}` |
| `POST` | `/search` | Flight search and aggregation |

```bash
curl http://localhost:8080/healthz
```

## Search

### One-way search

Returns all available flights from `origin` to `destination` on `departureDate`, normalized across all four providers.

| Param | Description |
|---|---|
| `origin` | Departure airport IATA code |
| `destination` | Arrival airport IATA code |
| `departureDate` | Date in `YYYY-MM-DD` format |
| `passengers` | Number of passengers (used to check seat availability) |
| `cabinClass` | `economy`, `business`, `first`, or `premium_economy` |

```bash
curl -s -X POST http://localhost:8080/search -H "content-type: application/json" -d "{\"origin\":\"CGK\",\"destination\":\"DPS\",\"departureDate\":\"2025-12-15\",\"passengers\":1,\"cabinClass\":\"economy\"}" | jq
```

### With filters and sort

Narrows results after aggregation and controls sort order — all filter/sort params are optional.

| Param | Description |
|---|---|
| `filters.max_price` | Drop flights above this IDR amount |
| `filters.max_stops` | `0` = direct only, `1` = max one stop, etc. |
| `filters.airlines` | Whitelist of airline IATA codes |
| `filters.departure_window` | Only flights departing between `from_hour` and `to_hour` (24h) |
| `sort_by` | `price`, `duration`, `departure_time`, `arrival_time`, or `best_value` |
| `sort_order` | `asc` or `desc` |

```bash
curl -s -X POST http://localhost:8080/search -H "content-type: application/json" -d "{\"origin\":\"CGK\",\"destination\":\"DPS\",\"departureDate\":\"2025-12-15\",\"passengers\":1,\"cabinClass\":\"economy\",\"filters\":{\"max_price\":1000000,\"max_stops\":0,\"airlines\":[\"GA\",\"JT\"],\"departure_window\":{\"from_hour\":6,\"to_hour\":12}},\"sort_by\":\"best_value\",\"sort_order\":\"asc\"}" | jq
```

### Round-trip

Same as one-way but also fetches the return leg — response contains both `outbound` and `inbound` flight arrays.

| Param | Description |
|---|---|
| `returnDate` | Return date in `YYYY-MM-DD` format |

```bash
curl -s -X POST http://localhost:8080/search -H "content-type: application/json" -d "{\"origin\":\"CGK\",\"destination\":\"DPS\",\"departureDate\":\"2025-12-15\",\"returnDate\":\"2025-12-20\",\"passengers\":1,\"cabinClass\":\"economy\"}" | jq
```

### POST /search — full parameter reference

#### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `origin` | string | yes | Departure airport IATA code (e.g. `"CGK"`) |
| `destination` | string | yes | Arrival airport IATA code (e.g. `"DPS"`) |
| `departureDate` | string | yes | Date in `YYYY-MM-DD` format |
| `returnDate` | string | no | Return date in `YYYY-MM-DD`. Omit or set `null` for one-way |
| `passengers` | int | yes | Number of passengers (must be ≥ 1) |
| `cabinClass` | string | yes | See cabin class options below. Defaults to `economy` if omitted |
| `filters` | object | no | See filters below. Omit the whole object to skip filtering |
| `sort_by` | string | no | See sort options below. Defaults to `price` |
| `sort_order` | string | no | `asc` or `desc`. Defaults to `asc` |

#### `cabinClass` options

| Value | Description |
|---|---|
| `economy` | Economy class |
| `premium_economy` | Premium economy |
| `business` | Business class |
| `first` | First class |

#### `filters` fields

All filter fields are optional. Omit any field to skip that filter.

| Field | Type | Description |
|---|---|---|
| `min_price` | int | Drop flights cheaper than this IDR amount |
| `max_price` | int | Drop flights more expensive than this IDR amount |
| `max_stops` | int | `0` = direct only, `1` = max one stop, `2` = max two stops, etc. |
| `airlines` | string[] | Whitelist by IATA code (e.g. `["GA", "JT"]`) or full name. Only matching airlines are returned |
| `max_duration_minutes` | int | Drop flights with total duration longer than this |
| `departure_window` | object | Only flights departing within the hour range — see below |
| `arrival_window` | object | Only flights arriving within the hour range — see below |

#### Time window fields (`departure_window` / `arrival_window`)

| Field | Type | Description |
|---|---|---|
| `from_hour` | int | Start of window, 24h format (0–23) |
| `to_hour` | int | End of window, 24h format (0–23). Wraps midnight if `to_hour` < `from_hour` |

#### `sort_by` options

| Value | Description |
|---|---|
| `price` | Sort by ticket price (default) |
| `duration` | Sort by total flight duration |
| `departure_time` | Sort by departure timestamp |
| `arrival_time` | Sort by arrival timestamp |
| `best_value` | Sort by composite score: price 60%, duration 30%, stops 10% |

## Tests

```bash
go test ./...
go test -race ./...
```

## Project Standards

Follows the [golang-standards/project-layout](https://github.com/golang-standards/project-layout) convention.
All doc comments follow the official [Go doc comment](https://go.dev/doc/comment) style.

## How the mock airline servers work

Each airline runs as a real HTTP server in the same process on a random port.
The providers call them over localhost HTTP — no shared memory, no function
calls. This mirrors a real multi-service architecture.

| Airline          | Simulated latency | Failure rate |
|------------------|-------------------|--------------|
| Garuda Indonesia | 50–100 ms         | none         |
| Lion Air         | 100–200 ms        | none         |
| Batik Air        | 200–400 ms        | none         |
| AirAsia          | 50–150 ms         | 10%          |

AirAsia failures return HTTP 503. The aggregator retries with exponential
backoff (2 attempts, full jitter), so transient failures are recovered
automatically.

## Design notes

**Separation of concerns.** Each layer has one job: providers handle fetching and normalizing, the aggregator handles fan-out and retries, filter/ranker handle result presentation, the service ties it together, and the API handler deals with HTTP. The goal was to make adding a new provider as isolated as possible — one file in `internal/pkg/providers/`, one handler in `internal/pkg/airlinemock/`, nothing else.

**Data inconsistency handling.** Each provider uses a different time format, which was one of the trickier parts. Garuda uses standard RFC3339, Batik Air drops the colon in the offset (`+0700`), AirAsia includes it (`+07:00`), and Lion Air gives naive datetime strings with a separate IANA timezone field. A normalizer per provider converts everything to RFC3339 + Unix timestamp. There's also a real data bug in Garuda's response — `GA315` has `arrival.airport: SUB` at the top level but the segments clearly end at `DPS`. The fix: always rebuild departure, arrival, and stop count from the segment chain when segments are present, and ignore the top-level fields. For duration, the provider-supplied fields (`duration_minutes`, `flight_time`, etc.) are also ignored and computed from the timestamps directly, since some of those values were off due to timezone mistakes in the source data. Each flight goes through a `Validate()` check before being included — if arrival isn't after departure, or price/duration are zero or negative, the flight is dropped.

**Parallelism + resilience.** All four providers are queried at the same time using goroutines and a `sync.WaitGroup`. Each runs under its own context with a 2 s timeout, so a slow one doesn't hold up the others. Retry logic with exponential backoff (up to 2 retries, full jitter) handles transient failures. AirAsia's 10% failure rate in the mock data gives this a real workout on every request.

**Rate limiting.** Token-bucket rate limiter per provider (20 rps, burst 10) to avoid hammering any single airline server when multiple search requests come in at the same time. Each limiter is shared across concurrent requests to that provider.

**Caching.** Raw aggregated results are cached before any filtering or sorting, keyed on `origin|destination|date|passengers|cabinClass|returnDate`. Filters and sort are deliberately left out of the cache key — two requests with different filters can share the same upstream data and just apply their own presentation on top. Default TTL is 30 s since flight prices change frequently.

**Dedup.** Multiple providers can return the same flight, so duplicates are collapsed by airline code + flight number + departure day, keeping the cheapest price. `timestamp / 86400` is used as the day key so it works correctly across timezone-aware timestamps.

**Best-value ranking.** The goal was a simple score that captures "cheap, fast, and direct" in one number. Price, duration, and stop count are normalized to a 0–1 scale relative to the current result set (cheapest/shortest/fewest = 1.0), then combined as a weighted sum: price 60%, duration 30%, stops 10%. The score is stored as `price.best_value_score` and available as a sort option. One side effect: scores shift when you filter results, since the min/max recalculates on the smaller set.

**IDR formatting.** Prices come back as both a raw integer and a formatted string like `"Rp 1.250.000"` — the raw integer is there so clients can still sort and compare without parsing the string.

## Bonus items

**Round-trip search.** When `returnDate` is provided, two separate searches run — one for the outbound leg and one for the return with origin and destination flipped. Each leg goes through its own full aggregation, dedup, cache, and presentation pass. The response wraps them as `outbound` and `inbound`.

**Timezone handling.** Lion Air's data was the tricky one — it gives naive datetime strings with no offset, and a separate field for the IANA timezone name. `time.ParseInLocation` combines them correctly. There's also a small airport lookup table that maps IATA codes to IANA timezone names (Jakarta → `Asia/Jakarta`, Denpasar → `Asia/Makassar`, Jayapura → `Asia/Jayapura`) as a fallback for any provider that doesn't supply a timezone at all.

**Per-provider rate limiting.** Each airline gets its own limiter so burst traffic from concurrent requests doesn't pile up on one provider.

**Exponential backoff with retry.** Failed calls are retried up to 2 times with full jitter backoff, capped at 1 s. The retry loop is tied to the per-provider context timeout, so it gives up as soon as the deadline hits rather than waiting for the full retry window.

**Parallel provider queries with timeout.** All four providers are queried at the same time. Each has an independent 2 s deadline, so the total response time is roughly the slowest single provider rather than the sum of all four.

**Graceful shutdown.** On `SIGINT` or `SIGTERM` the server calls `srv.Close()` so in-flight requests finish before the process exits.

**Not implemented:** multi-city search. The requirement was marked optional and it would need a different request/response shape. The current single-leg model could support it as a sequence of one-way calls.

## Sample response

```bash
curl -s -X POST http://localhost:8080/search -H "content-type: application/json" -d "{\"origin\":\"CGK\",\"destination\":\"DPS\",\"departureDate\":\"2025-12-15\",\"passengers\":1,\"cabinClass\":\"economy\"}" | jq
```

One-way search CGK → DPS, 13 flights returned (one shown for simplicity).

```json
{
  "search_criteria": {
    "origin": "CGK",
    "destination": "DPS",
    "departure_date": "2025-12-15",
    "passengers": 1,
    "cabin_class": "economy"
  },
  "metadata": {
    "total_results": 13,
    "providers_queried": 4,
    "providers_succeeded": 4,
    "providers_failed": 0,
    "search_time_ms": 311,
    "cache_hit": false
  },
  "flights": [
    {
      "id": "QZ532_AirAsia",
      "provider": "AirAsia",
      "airline": {
        "name": "AirAsia",
        "code": "QZ"
      },
      "flight_number": "QZ532",
      "departure": {
        "airport": "CGK",
        "city": "Jakarta",
        "datetime": "2025-12-15T19:30:00+07:00",
        "timestamp": 1765801800
      },
      "arrival": {
        "airport": "DPS",
        "city": "Denpasar",
        "datetime": "2025-12-15T22:10:00+08:00",
        "timestamp": 1765807800
      },
      "duration": {
        "total_minutes": 100,
        "formatted": "1h 40m"
      },
      "stops": 0,
      "price": {
        "amount": 595000,
        "currency": "IDR",
        "formatted": "Rp 595.000",
        "best_value_score": 0.9516
      },
      "available_seats": 72,
      "cabin_class": "economy",
      "aircraft": null,
      "amenities": [],
      "baggage": {
        "carry_on": "Cabin baggage only",
        "checked": "Additional fee"
      }
    }
  ]
}
```