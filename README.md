# splitwise-cli

A zero-dependency Python CLI for [Splitwise](https://www.splitwise.com) using the **web session cookie** to hit the internal `v3.0` API directly — no OAuth app registration, no API key, works on free accounts (the official OAuth API restricts several endpoints like `get_main_data`, but your own web session can reach them).

Stdlib only (`urllib`, `json`, `argparse`). Single file. ~280 lines.

## Why

Splitwise's official developer API requires an approved OAuth app, and even then:
- writes (`create_expense`, payments) are limited/forbidden for many accounts,
- the balances the dashboard shows you are computed server-side and only exposed to the web app itself.

This CLI authenticates the same way the web app does — your `_splitwise_session` cookie — and reproduces exactly what the dashboard renders. Reads **and** writes (expenses, custom splits, payments) work.

## Setup

1. Log in to Splitwise in your browser.
2. DevTools → Application → Cookies → `https://secure.splitwise.com` → copy `name=value` pairs (at minimum `_splitwise_session`; grabbing `remember_token` + others is fine).
3. Save them, one per line, to `~/finance/splitwise_cookies.txt` (path configurable via `COOKIE_FILE` at the top of the script):

   ```
   _splitwise_session=eyJ...
   remember_token=...
   ```

4. `chmod 600` the cookie file. **Never commit it.**

Session cookies expire after some weeks — if the CLI starts returning HTML instead of JSON, re-copy the cookie.

## Commands

```
splitwise me                          # who am I (id, name, email)
splitwise groups                      # id — name per group
splitwise friends                     # id, names, emails
splitwise balances                    # ← the trustworthy one, see below
splitwise expenses [--group ID] [--limit N] [--details]
splitwise add --desc "dinner" --total 60 [--currency MYR]
          [--group ID | --with Alice,Bob]
          [--split equal | --split shares --shares "Alice=20,Bob=15"]
          [--date YYYY-MM-DD] [--dry-run]
splitwise settle --with Alice --amount 20 [--group ID] [--dry-run]
splitwise delete EXPENSE_ID
```

`add` defaults to you paying the full amount; `--split shares` takes explicit owed shares per person (remainder falls to you). `settle` records a payment (negative expense) so balances move.

## Balances: the one rule that matters

**Do not reconstruct balances by summing `net_balance` across expense rows.** Two ways this looks right and is wrong:

1. `net_balance` on each `get_expenses` user row is a **per-expense delta**, not a running total. Summing it multi-counts; taking only the latest row under-counts.
2. Group expenses settle at the **group** level. Summing your shares across Penang-group + Danang-group expenses does *not* give you a per-friend balance — payments in one scope don't transfer to another.

The correct source is `GET /api/v3.0/get_main_data` — the exact payload the web dashboard renders. `friends[].balance` and `groups[].balance` are **precomputed server-side**. Sign convention (verified against the app's own source): **positive = they owe you** ("gets back"), negative = you owe them. An empty balance array = settled.

`splitwise balances` reads only those arrays. Zero client-side maths.

## API notes (v3.0, session-cookie auth)

- Base: `https://secure.splitwise.com/api/v3.0`
- Reads: cookie + `Accept: application/json` is enough.
- Writes (`create_expense`, `delete_expense`): also need header `X-CSRF-Token` (scraped from `name="csrf-token"` in the dashboard HTML) + `Origin`/`Referer` headers, and **JSON body with flat keys**:

  ```json
  {
    "description": "lunch", "cost": "60.00", "currency_code": "MYR",
    "date": "2026-10-13T12:00:00Z", "group_id": 0,
    "users__0__user_id": 123, "users__0__paid_share": "60.00", "users__0__owed_share": "0.00",
    "users__1__user_id": 456, "users__1__paid_share": "0.00",  "users__1__owed_share": "60.00"
  }
  ```

  The nested-object form (`{"expense": {...}}`) is rejected with *Unrecognized parameter `expense`*. Payments are just `create_expense` with `"payment": true`.
- `split_equally: true` works for group expenses; for non-group (friend) expenses you must send explicit `paid/owed_share` per user even for equal splits.
- `get_group?id=N` (query param, not path) returns members; `get_group_balance` does not exist.
- Errors come back as `{"errors": {"base": ["..."]}}` with HTTP 400/403.

## Install

```sh
mkdir -p ~/.local/bin
curl -fsSL -o ~/.local/bin/splitwise https://raw.githubusercontent.com/shuenrui/splitwise-cli/main/splitwise
chmod +x ~/.local/bin/splitwise
```

## License

MIT
