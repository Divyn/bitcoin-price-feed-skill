# Bitcoin Price Feed Skill

Real-time streaming Bitcoin price feed for traders. Subscribe to a live Bitcoin price stream over WebSocket: OHLC ticks, volume, and derived metrics (moving averages, % change) streamed in real time from the [Bitquery](https://bitquery.io) GraphQL API.

## Features

- **Live OHLC** — Open, High, Low, Close for 1-second intervals
- **Volume (USD)** — Per-tick volume
- **Derived metrics** — Mean, SMA, EMA, WMA from the API; optional tick-to-tick % change on the stream
- **WebSocket** — No polling; data streams as it arrives

## Prerequisites

- Python 3.8+
- A [Bitquery](https://account.bitquery.io/user/api_v2/access_tokens) OAuth key (required for the streaming endpoint)

## Setup

1. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

   Or directly:

   ```bash
   pip install 'gql[websockets]'
   ```

2. **Set your API key**

   ```bash
   export BITQUERY_API_KEY=your_bitquery_token
   ```

   The script will not run without this; it will exit with a clear error if the key is missing.

## Usage

### Run the streaming script

Stream Bitcoin ticks until you stop it (Ctrl+C) or until an optional timeout:

```bash
python scripts/stream_bitquery.py
```

Stop after 60 seconds:

```bash
python scripts/stream_bitquery.py --timeout 60
```

### Inline subscription (Python)

You can also subscribe in your own code:

```python
import asyncio
import os
from gql import Client, gql
from gql.transport.websockets import WebsocketsTransport

async def main():
    token = os.environ["BITQUERY_API_KEY"]
    url = f"wss://streaming.bitquery.io/graphql?token={token}"
    transport = WebsocketsTransport(
        url=url,
        headers={"Sec-WebSocket-Protocol": "graphql-ws"},
    )
    async with Client(transport=transport) as session:
        sub = gql("""
            subscription {
                Trading {
                    Tokens(where: {Currency: {Id: {is: "bid:bitcoin"}}, Interval: {Time: {Duration: {eq: 1}}}}) {
                        Token { Name Symbol Network }
                        Block { Time }
                        Price { Ohlc { Open High Low Close } Average { Mean SimpleMoving ExponentialMoving } }
                        Volume { Usd }
                    }
                }
            }
        """)
        async for result in session.subscribe(sub):
            print(result)

asyncio.run(main())
```

## Output

Each tick includes:

- **OHLC** and **Volume (USD)** for the 1-second interval
- **Derived metrics**: Mean, SimpleMoving (SMA), ExponentialMoving (EMA), WeightedSimpleMoving (WMA)
- **Session-derived**: % change vs previous tick (when using the script)

Example formatted tick:

```
Bitcoin (BTC) — ethereum network  @ 2025-03-06T14:00:00Z

OHLC:
  Open:  $85,200.00  High: $86,100.00  Low: $84,950.00  Close: $85,780.00
  Derived (on stream):
    Mean:   $85,500.00   SMA: $85,400.00   EMA: $85,520.00
  Tick Δ: +0.12% vs previous

Volume (USD): $1,234,567.00
```

## Intervals

The default subscription uses **duration 1** (1-second tick data). The same `Trading.Tokens` subscription supports other durations in the `where` clause (e.g. 5, 60, 1440 for 5m, 1h, 1d candles) if supported by the API for subscriptions.

## Error handling

| Situation | Action |
|-----------|--------|
| **Missing BITQUERY_API_KEY** | Script exits; tell user to export the variable |
| **WebSocket connection failed / 401** | Token invalid or expired |
| **Subscription errors in payload** | Log error, close transport cleanly |
| **No ticks received** | Check token and network; first tick may be delayed |

## Project structure

```
bitquery-bitcoin-price-skill/
├── README.md           # This file
├── SKILL.md            # Skill instructions for the agent
├── requirements.txt    # gql[websockets]
├── scripts/
│   └── stream_bitquery.py   # WebSocket streaming script
├── references/
│   └── graphql-fields.md   # Trading.Tokens field reference
└── evals/
    └── evals.json      # Evaluation prompts and expected behavior
```

## Reference

- **GraphQL fields**: See `references/graphql-fields.md` for filters, fields, and subscription structure (e.g. date range, other durations).
- **Bitquery**: [streaming.bitquery.io](https://streaming.bitquery.io) — WebSocket endpoint used by this skill.
