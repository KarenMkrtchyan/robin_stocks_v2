# Robinhood Integration Guide

A practical reference for integrating `robin_stocks.robinhood` into a larger application that coordinates multiple brokerages. This guide covers authentication, session persistence, MFA handling, placing trades, and monitoring balances across multiple Robinhood accounts under one user login.

For the complete API surface (all functions, parameters, and return shapes) see [`Robinhood.rst`](Robinhood.rst) or the [readthedocs site](https://robin-stocks.readthedocs.io/en/latest/robinhood.html).

---

## Installation

```bash
pip install -e ".[mcp,dev]"   # development install with test/MCP extras
# or for just the SDK:
pip install robin_stocks
```

---

## Quick Start

```python
import robin_stocks.robinhood as r

r.login("your@email.com", "yourpassword")

profile = r.build_user_profile()
print(profile)
# {'equity': '12345.67', 'extended_hours_equity': '12300.00', 'cash': '500.00', 'dividend_total': '42.10'}
```

After `login` succeeds, every subsequent call in the process automatically uses the stored session—no token passing required.

---

## Login

### Function signature

```python
r.login(
    username=None,        # Prompts via input() if omitted
    password=None,        # Prompts via getpass.getpass() if omitted
    expiresIn=86400,      # Token lifetime in seconds (default: 1 day)
    scope="internal",     # OAuth scope
    store_session=True,   # Write/reuse ~/.tokens/robinhood.json
    mfa_code=None,        # Pre-computed TOTP code (see MFA section)
    pickle_path="",       # Override session file directory
    pickle_name="",       # Suffix for the session filename
)
```

### Parameters in detail

| Parameter | Default | Notes |
|-----------|---------|-------|
| `username`, `password` | `None` (interactive prompt) | Pass explicitly for automation; use env vars, not literals |
| `mfa_code` | `None` | A 6-digit TOTP code. Required only if you've enrolled TOTP with Robinhood's "Other" authenticator option. See [MFA section](#mfa--device-verification) |
| `store_session` | `True` | Persist credentials to disk and reuse them on the next call. Set `False` to force a fresh login every time and skip writing a file |
| `pickle_path` | `""` → `~/.tokens` | Directory where the session JSON is written. Despite the parameter name the file format is JSON, not pickle |
| `pickle_name` | `""` | Appended to the filename: `robinhood{pickle_name}.json`. Use this when running multiple Robinhood accounts from the same machine |
| `expiresIn` | `86400` | Only affects the OAuth `expires_in` field sent to Robinhood; the SDK does not auto-refresh on expiry |

### Return value

`login` returns a dictionary on success or `None` on failure.

**Fresh login (new OAuth token):**
```python
{
    "access_token":  "...",
    "token_type":    "Bearer",
    "expires_in":    86400,
    "scope":         "internal",
    "refresh_token": "...",
    # plus any other fields Robinhood includes
}
```

**Cached session (session file was valid):**
```python
{
    "access_token":  "...",
    "token_type":    "Bearer",
    "expires_in":    86400,
    "scope":         "internal",
    "refresh_token": "...",
    "backup_code":   None,
    "detail":        "logged in using authentication in robinhood.json"
}
```

**Failure:** `None`, with an error message printed to stdout.

### Recommended pattern for automation

Store credentials in environment variables (or a `.env` file loaded with `python-dotenv`):

```python
import os
from dotenv import load_dotenv
import robin_stocks.robinhood as r

load_dotenv()  # reads .env from the working directory

r.login(
    username=os.environ["ROBINHOOD_USERNAME"],
    password=os.environ["ROBINHOOD_PASSWORD"],
    store_session=True,
)
```

Never hardcode credentials in source files. See [`examples/robinhood examples/two_factor_log_in.py`](examples/robinhood%20examples/two_factor_log_in.py) for the full dotenv pattern.

---

## MFA & Device Verification

Robinhood enforces two separate security mechanisms. They can trigger independently or together.

### Path 1: TOTP ("Other" authenticator app)

This is a static TOTP secret you register once in the Robinhood app:

1. In the Robinhood app, go to **Account → Security → Two-Factor Authentication** and select **Other** when asked which app to use.
2. Robinhood shows you an alphanumeric secret (e.g. `JBSWY3DPEHPK3PXP`). Copy it and store it securely — this is your `mfa_secret`. Also save the backup code Robinhood displays.
3. Enter the secret into your authenticator app (e.g. Google Authenticator) to verify it works.

With that secret, generate the current 6-digit code at login time using `pyotp`:

```python
import pyotp
import os
import robin_stocks.robinhood as r

totp = pyotp.TOTP(os.environ["ROBINHOOD_MFA_SECRET"]).now()
r.login(
    username=os.environ["ROBINHOOD_USERNAME"],
    password=os.environ["ROBINHOOD_PASSWORD"],
    mfa_code=totp,
    store_session=True,
)
```

The TOTP code is included directly in the OAuth payload so login completes in one round-trip when the token is valid.

### Path 2: Device verification ("sheriff" workflow)

Even with `mfa_code` supplied, Robinhood may respond with a `verification_workflow` field in the OAuth response — this is what triggered the phone notification you saw when running `simple_test.py`. The SDK handles it automatically inside `_validate_sherrif_id`, but your application needs to be aware of the user-facing behavior:

| Challenge type | What happens in your process |
|----------------|------------------------------|
| `prompt` | Prints `"Check robinhood app for device approvals..."` and polls the Robinhood API every 5 seconds for up to ~2 minutes until the user taps Approve in the app |
| `sms` / `email` | Blocks on `input()` waiting for the user to paste the verification code sent to their phone or email |

After the challenge is resolved the SDK re-posts the original login request and continues normally.

**Implication for background services:** the sheriff `prompt` path only needs a phone tap (non-blocking for your terminal process), but `sms`/`email` requires an interactive terminal. If your service needs fully headless operation, use `store_session=True` and perform the first login interactively so the session file is created. Subsequent runs will skip the OAuth POST entirely if the cached token is still valid.

---

## Session Files

### What gets stored

After a successful fresh login (with `store_session=True`), the SDK writes:

```
~/.tokens/robinhood.json
```

The file contains plaintext JSON — no encryption:

```json
{
  "token_type": "Bearer",
  "access_token": "<oauth-access-token>",
  "refresh_token": "<oauth-refresh-token>",
  "device_token": "<uuid-device-identifier>"
}
```

### How reuse works

Every subsequent `login(...)` call checks for this file first:

1. Load `access_token` and `device_token` from the file.
2. Set the `Authorization` header on the global session object.
3. Send a probe `GET /positions/` — if it returns 200 the SDK sets `LOGGED_IN = True` and returns immediately (no network auth round-trip, no MFA challenge).
4. If the probe fails for any reason (expired token, network error, corrupt file), the SDK logs an error and falls through to a full fresh login.

### Important caveats for integrators

**Tokens are not auto-refreshed.** The `refresh_token` is stored in the file but the SDK does not use it to silently renew the `access_token`. When the token expires (default: 24 hours), the next `login` call that fails the probe will do a full re-login — which may trigger MFA again.

**One active session per process.** The SDK uses a single global `requests.Session` object and a single `LOGGED_IN` flag per Python process. If you need to manage two separate Robinhood user accounts simultaneously, use separate OS processes (e.g. subprocesses or separate workers) each with their own `pickle_name`, so their session files don't collide.

**`logout()` does not delete the session file.** It clears `LOGGED_IN` and removes the `Authorization` header from the in-memory session. The JSON file on disk is left untouched.

### Customizing the file location

```python
# Specific directory and account-specific filename
r.login(
    username=os.environ["ROBINHOOD_USERNAME"],
    password=os.environ["ROBINHOOD_PASSWORD"],
    pickle_path="/var/secrets/tokens",   # absolute or relative path
    pickle_name="_account_A",            # writes robinhood_account_A.json
)
```

### Security

Treat `~/.tokens/robinhood.json` with the same care as a password:

- Add `~/.tokens/` to `.gitignore`.
- Restrict file permissions: `chmod 600 ~/.tokens/robinhood.json`.
- Never log or print the file contents.

---

## Placing Trades

All order functions require an active login. Calling them before `login()` raises an exception immediately via the `@login_required` decorator.

### Market buy and sell

```python
import robin_stocks.robinhood as r

r.login(...)

# Buy 10 shares of AAPL at market price
order = r.order_buy_market("AAPL", 10)
print(order["state"])   # "queued", "confirmed", "filled", "failed", etc.

# Sell 5 shares of TSLA at market price
r.order_sell_market("TSLA", 5)
```

### Specifying an account

All stock order functions accept an optional `account_number` parameter. Omitting it sends the order to the first account Robinhood returns (see the [Multiple Accounts](#multiple-robinhood-accounts) section for how to enumerate them):

```python
r.order_buy_market("AAPL", 10, account_number="XXXXXXXX")
```

### Checking the order response

By default the functions return a parsed dict. Pass `jsonify=False` to get the raw `requests.Response` object — useful for checking HTTP status codes and implementing retry logic:

```python
from time import sleep

order = r.order_buy_market("AAPL", 10, jsonify=False)

attempts = 0
while order.status_code != 200 and attempts < 5:
    sleep(2)
    order = r.order_buy_market("AAPL", 10, jsonify=False)
    attempts += 1

if order.status_code == 200:
    print("Order placed:", order.json())
else:
    print("Failed:", order.json().get("detail"))
```

See [`examples/robinhood examples/auto_retry_order.py`](examples/robinhood%20examples/auto_retry_order.py) for the complete retry pattern.

### Other order types

For limit orders, stop orders, fractional shares, options, and crypto, see [`Robinhood.rst`](Robinhood.rst) "Placing Orders" or the [readthedocs page](https://robin-stocks.readthedocs.io/en/latest/robinhood.html).

---

## Multiple Robinhood Accounts

A single Robinhood login can have several accounts (e.g. individual brokerage, IRA, margin). By default most SDK functions operate on the **first** account Robinhood returns. To work with a specific account, discover its `account_number` and pass it explicitly.

### List all accounts

```python
import robin_stocks.robinhood as r
from robin_stocks.robinhood.helper import request_get
from robin_stocks.robinhood.urls import account_profile_url

r.login(...)

# account_profile_url() with no argument returns the "all accounts" endpoint.
# "pagination" collects every page of results into a single list.
accounts = request_get(account_profile_url(), "pagination")

for acct in accounts:
    print(acct["account_number"], acct["type"], acct["state"])
```

### Per-account balance snapshot

```python
for acct in accounts:
    num = acct["account_number"]

    cash   = r.load_account_profile(account_number=num, info="portfolio_cash")
    equity = r.load_portfolio_profile(account_number=num, info="equity")
    buying = r.load_account_profile(account_number=num, info="buying_power")

    print(f"Account {num}: equity={equity}  cash={cash}  buying_power={buying}")
```

### Comprehensive profile per account

`build_user_profile` rolls equity, extended-hours equity, cash, and total dividends into one call:

```python
for acct in accounts:
    profile = r.build_user_profile(account_number=acct["account_number"])
    # {'equity': '...', 'extended_hours_equity': '...', 'cash': '...', 'dividend_total': '...'}
    print(acct["account_number"], profile)
```

### Open positions per account

```python
for acct in accounts:
    positions = r.get_open_stock_positions(account_number=acct["account_number"])
    for pos in positions:
        symbol = r.get_symbol_by_url(pos["instrument"])
        print(f"{acct['account_number']}  {symbol}  qty={pos['quantity']}")
```

---

## Monitoring After Trades

After placing an order you typically want to poll its status and watch account-level figures change as it fills.

### Check open orders

```python
open_orders = r.get_all_open_stock_orders()
for order in open_orders:
    print(order["id"], order["side"], order["quantity"], order["state"])
```

For a specific account:

```python
from robin_stocks.robinhood.helper import request_get
from robin_stocks.robinhood.urls import orders_url

open_orders = request_get(orders_url(account_number="XXXXXXXX"), "pagination")
```

### Key balance fields from `load_account_profile`

| Field | Meaning |
|-------|---------|
| `buying_power` | Cash available for new trades right now |
| `portfolio_cash` | Total cash in the account |
| `cash_held_for_orders` | Cash reserved for pending buy orders |
| `unsettled_funds` | Funds from recent sells not yet settled (T+1) |
| `equity` (from `load_portfolio_profile`) | Total market value including open positions |

### Quick post-trade poll loop

```python
import time
import robin_stocks.robinhood as r

account_number = "XXXXXXXX"

for _ in range(12):   # poll for up to ~1 minute
    open_orders = r.get_all_open_stock_orders()
    if not open_orders:
        break
    print(f"{len(open_orders)} order(s) still open...")
    time.sleep(5)

profile = r.build_user_profile(account_number=account_number)
print("Final equity:", profile["equity"])
print("Cash:        ", profile["cash"])
```

---

## Further Reading

- [`Robinhood.rst`](Robinhood.rst) — complete list of all functions with parameters and example calls.
- [`examples/robinhood examples/`](examples/robinhood%20examples/) — runnable scripts: dotenv login, TOTP login, auto-retry orders, fund deposits/withdrawals.
- [readthedocs API reference](https://robin-stocks.readthedocs.io/en/latest/robinhood.html) — fully rendered docs with all return value keys documented.
- [`robin_stocks/robinhood/authentication.py`](robin_stocks/robinhood/authentication.py) — the login implementation: session cache logic, `_validate_sherrif_id`, token storage.
