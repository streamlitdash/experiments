# Fix 1: Market calls

This guide explains only the Market-call fix introduced by commit `3d1ef5e`.
The implementation is in `rebirth/services/s06_refresh.py`.

## Problem

Open and Current Market data used to load one Underlying at a time. One slow or
temporary failed call could therefore delay the entire refresh.

## Current fix

The file defines three settings:

```python
_MARKET_MAX_WORKERS = 20
_MARKET_RETRIES = 4
_MARKET_RETRY_DELAY_SECONDS = 0.5
```

For each product:

1. `_load_product_market_open()` prepares the Open connector call.
2. `_load_product_market_status()` prepares the Current connector call.
3. Both pass their ordered Underlying list to `_load_market_frames()`.
4. `_load_market_frames()` runs at most 20 calls concurrently.
5. A failed call is retried four times after its first attempt, giving five
   possible attempts in total.
6. Results are returned in the original Underlying order and concatenated only
   after every call succeeds.

`TypeError` and `ValueError` are not retried because they normally indicate bad
connector code or an invalid schema. Other connector exceptions are retried.

If every retry fails, the refresh fails without publishing a partial snapshot.
The application retains its last good snapshot.

## Add a real timeout

The thread pool lets other calls continue while one call is slow, but it cannot
safely kill a permanently stuck Python function. Put the timeout on the real
MRX or network call itself.

Example:

```python
MARKET_TIMEOUT_SECONDS = 5


def get_market_data(market_date, underlying, *, market_status):
    try:
        return mrx_call(
            market_date,
            underlying,
            market_status=market_status,
            timeout=MARKET_TIMEOUT_SECONDS,
        )
    except TimeoutError as exc:
        raise TimeoutError(
            f"Market call timed out for {underlying}"
        ) from exc
```

Replace `timeout` with the actual timeout argument supported by the MRX client.
If it expects milliseconds, use `5_000` instead of `5`.

No refresh-manager change is needed after adding this timeout. The timeout
exception reaches `_load_market_frames()`, which performs the four retries.

Do not use `future.result(timeout=5)` as a hard-stop replacement. It can stop
the caller waiting, but it does not terminate the underlying stuck thread.

## Adjust the settings

Change only the constants near the top of
`rebirth/services/s06_refresh.py`:

```python
_MARKET_MAX_WORKERS = 20          # simultaneous Market calls
_MARKET_RETRIES = 4               # retries after the first attempt
_MARKET_RETRY_DELAY_SECONDS = 0.5 # wait before each retry
```

Keep the worker count within the connection limit allowed by the real Market
service.

## Verify the fix

Run the focused test:

```powershell
python -m pytest tests/s07_integration.py -q
```

The tests confirm that Open and Current calls overlap, results preserve their
original order, and a temporary failure succeeds after four retries.

To remove the code change completely, revert commit `3d1ef5e`.
