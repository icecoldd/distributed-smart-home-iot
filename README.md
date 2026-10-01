# Distributed Smart-Home IoT System

A collaborative coursework prototype for querying sensor readings across two smart homes using **Python, TCP sockets, and PostgreSQL/Neon**.

This portfolio fork preserves the original history of [cobyxa/assign8](https://github.com/cobyxa/assign8). **Ryan Carrasco and Coby Xa** developed the project together. Ryan's client implementation is recorded in commit `c89076e`; the server commit `075de1c` explicitly credits both Coby and Ryan.

## What the code demonstrates

- A persistent terminal menu with three queries: average fridge moisture, average dishwasher water readings, and a 24-hour electricity comparison.
- TCP client-to-server requests and JSON server-to-server `FETCH` requests.
- Parameterized PostgreSQL queries against an existing `clean_iot_data` relation.
- Combining local and remote sums and counts to calculate a weighted mean, rather than averaging averages.

```text
Client A -> Server A <------ TCP / JSON ------> Server B <- Client B
               |                                  |
         PostgreSQL A                       PostgreSQL B
```

Each server accesses its configured database. Remote aggregate reads go through the partner server; no second database connection string is used.

## Distributed-data assumptions

The prototype assumes each house has its own historical readings before `SHARING_START`, and that both houses' newer readings are available locally after that point. For average queries starting before the cutoff, it adds the partner's historical sum/count to the local result. This is correct only when the assumed coverage and non-overlap hold.

**The repository does not implement ingestion, replication, or coverage detection.** The cutoff is a configured timestamp, not a test of whether records are actually present. Electricity comparison reads only the local database and requires both houses' readings to already be there.

## Files

| File | Purpose |
| --- | --- |
| `client.py` | Interactive TCP client and query menu |
| `server.py` | SQL aggregation, peer requests, and TCP server |

The original SQL definition for `clean_iot_data`, DataNiz configuration, deployment scripts, and sample dataset are not included. The Python code expects these columns:

| Column | Expected use |
| --- | --- |
| `house_id` | `A` or `B` for electricity comparison |
| `metric` | `moisture`, `water`, or `electricity` |
| `device_type` | `fridge` or `dishwasher` for average queries |
| `value` | Numeric sensor reading |
| `timestamp` | Sensor reading time |

The code averages dishwasher **readings**; interpreting that as an average per cycle requires one valid reading per cycle. Electricity totals likewise require additive readings with consistent units.

## Running the original prototype

Python 3.10+ and an existing compatible PostgreSQL database are required. This is not yet a self-contained demo.

```sh
python -m venv .venv
source .venv/bin/activate
pip install psycopg2-binary
```

On each server, set the local `DB_URL`, the other server's `PARTNER_HOST` / `PARTNER_PORT`, and a `SHARING_START` that matches the dataset. Keep real credentials out of commits. The original source uses placeholders.

Start each server, choosing its listening port:

```sh
python server.py
```

Then connect the client to the desired server address and port:

```sh
python client.py
```

For a local two-server setup, use separate terminal processes and different listening ports, with each partner port pointing to the other server. Database contents must follow the coverage assumptions above.

## Current limitations

This fork presents the original coursework implementation, with documentation added for portfolio review. It is not production-ready.

- The server handles one connection at a time. An idle persistent client prevents it from answering another client's request or a partner `FETCH`; two occupied servers can block each other.
- TCP messages have no framing and each receive reads at most 4096 bytes. Partial messages are not reassembled, and socket timeouts are not configured.
- Missing averages are returned as text but formatted as numbers, resulting in `Invalid query`.
- PostgreSQL `numeric` aggregates may return Python `Decimal` values, which the current JSON response cannot serialize. The exception handler returns zero aggregates.
- Some database/partner failures are converted into zero aggregates, allowing incomplete results to look successful.
- Electricity ties (including no rows) incorrectly report House B as the winner, and missing house data is treated as zero.
- The TCP protocol has no authentication or TLS. Use an isolated development environment rather than exposing these ports broadly.

A reproducible portfolio demo would next need the original cleaning SQL (or an explicitly labeled replacement), synthetic data, concurrent connection handling, framed messages, explicit error responses, and regression tests.
