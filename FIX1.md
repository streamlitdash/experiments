# Fix 1: implement concurrent Market calls with retries

This is the complete step-by-step implementation guide for Fix 1. It changes
only Market Open and Market Current loading. Risk loading remains sequential
and unchanged.

The completed implementation is also available in Rebirth V4.1 commit
`3d1ef5e`.

## What the fix does

- Runs at most 20 Market calls at the same time.
- Applies the same behavior to Open and Current Market data.
- Retries a failed call four times after its first attempt.
- Waits 0.5 seconds before each retry.
- Keeps returned frames in the original Underlying order.
- Validates that every result is a pandas `DataFrame`.
- Does not retry `TypeError` or `ValueError`, because these normally indicate a
  coding or schema problem.
- Does not publish a partial refresh if a Market call ultimately fails.

With 370 Credit Underlyings, the first 20 calls start, then the next call starts
whenever one finishes. The code does not create 370 simultaneous calls.

## Files to change

Only these two files are required:

```text
rebirth/services/s06_refresh.py
tests/s07_integration.py
```

Run every command below from the Rebirth V4.1 repository root.

## Step 1: import the thread pool

Open `rebirth/services/s06_refresh.py`.

Find the imports at the top of the file. Add this import:

```python
from concurrent.futures import ThreadPoolExecutor
```

The opening imports should now include it:

```python
import logging
import re
import time
import uuid
from concurrent.futures import ThreadPoolExecutor
from dataclasses import replace
```

No new package is required. `ThreadPoolExecutor` is part of Python.

## Step 2: add the three settings

In the same file, find:

```python
LOGGER = logging.getLogger("rebirth.services.s06_refresh")
```

Directly underneath it, add:

```python
_MARKET_MAX_WORKERS = 20
_MARKET_RETRIES = 4
_MARKET_RETRY_DELAY_SECONDS = 0.5
```

The meanings are:

- `_MARKET_MAX_WORKERS = 20`: no more than 20 calls run simultaneously.
- `_MARKET_RETRIES = 4`: four retries after the initial call, so five total
  attempts are possible.
- `_MARKET_RETRY_DELAY_SECONDS = 0.5`: wait half a second between attempts.

## Step 3: add the shared Market loader

Inside `RiskRefreshManager`, find `_load_product_risk()`.

Insert the following method immediately after `_load_product_risk()` and before
`_load_product_market_open()`:

```python
    def _load_market_frames(
        self,
        spec: ProductSpec,
        underlyings: tuple[str, ...],
        *,
        connector: Callable[..., pd.DataFrame],
        stage: str,
        label: str,
        load_one: Callable[[str], object],
    ) -> pd.DataFrame:
        """Load one product's market scope concurrently in stable input order."""
        connector_name = _callable_name(connector)
        failure_message = (
            "Opening market connector failed."
            if stage == "market_open"
            else "Current market connector failed."
        )

        def load_with_retry(item: tuple[int, str]) -> pd.DataFrame:
            index, underlying = item
            for retry in range(_MARKET_RETRIES + 1):
                retry_message = (
                    ""
                    if retry == 0
                    else f" Retry {retry} of {_MARKET_RETRIES}."
                )
                self._progress_activity(
                    connector_name,
                    stage,
                    source_type=spec.source_type,
                    underlying=underlying,
                    product_index=index,
                    product_total=len(underlyings),
                    message=f"Loading {label} for {underlying}.{retry_message}",
                )
                try:
                    frame = load_one(underlying)
                except (TypeError, ValueError):
                    self._progress_activity(
                        connector_name,
                        stage,
                        source_type=spec.source_type,
                        underlying=underlying,
                        message=failure_message,
                    )
                    raise
                except Exception as exc:
                    if retry == _MARKET_RETRIES:
                        self._progress_activity(
                            connector_name,
                            stage,
                            source_type=spec.source_type,
                            underlying=underlying,
                            message=failure_message,
                        )
                        raise
                    LOGGER.warning(
                        "Market connector failed for %s/%s; retry %d of %d: %s",
                        spec.source_type,
                        underlying,
                        retry + 1,
                        _MARKET_RETRIES,
                        exc,
                    )
                    self._sleep(_MARKET_RETRY_DELAY_SECONDS)
                    continue
                if not isinstance(frame, pd.DataFrame):
                    kind = "market Open" if stage == "market_open" else "current market"
                    raise TypeError(f"{kind} connector must return a pandas DataFrame")
                return frame
            raise AssertionError("unreachable market retry state")

        with ThreadPoolExecutor(max_workers=_MARKET_MAX_WORKERS) as executor:
            frames = list(
                executor.map(load_with_retry, enumerate(underlyings, start=1))
            )
        return pd.concat(frames, ignore_index=True, sort=False)
```

Why `executor.map()` is used: calls may finish in any order, but `map()` returns
their results in the same order as `underlyings`. This keeps downstream output
deterministic.

## Step 4: replace the Open Market method

Still in `rebirth/services/s06_refresh.py`, replace the complete existing
`_load_product_market_open()` method with:

```python
    def _load_product_market_open(
        self,
        spec: ProductSpec,
        open_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame:
        # Every Open adapter receives the authoritative T-1 business date.
        adapter = self._connector_adapters.get(spec.source_type)
        connector = (
            adapter.market_open if adapter is not None else self._market_open_loader
        )
        selected_status = _require_market_status(market_status)

        def load_one(underlying: str) -> object:
            if adapter is not None:
                return adapter.market_open(
                    open_date, underlying, market_status=selected_status
                )
            return self._market_open_loader(
                spec.source_type,
                open_date,
                underlying,
                market_status=selected_status,
            )

        return self._load_market_frames(
            spec,
            underlyings,
            connector=connector,
            stage="market_open",
            label="Open",
            load_one=load_one,
        )
```

The adapter still receives exactly one Underlying per call. The only change is
that several independent calls can now run together.

## Step 5: replace the Current Market method

Replace the complete existing `_load_product_market_status()` method with:

```python
    def _load_product_market_status(
        self,
        spec: ProductSpec,
        market_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame:
        adapter = self._connector_adapters.get(spec.source_type)
        connector = (
            adapter.market_status if adapter is not None else self._market_status_loader
        )
        selected_status = _require_market_status(market_status)

        def load_one(underlying: str) -> object:
            if adapter is not None:
                return adapter.market_status(
                    market_date, underlying, market_status=selected_status
                )
            return self._market_status_loader(
                spec.source_type,
                market_date,
                underlying,
                market_status=selected_status,
            )

        return self._load_market_frames(
            spec,
            underlyings,
            connector=connector,
            stage="market_status",
            label=selected_status,
            load_one=load_one,
        )
```

Do not alter `_load_product_risk()`. Fix 1 intentionally changes only Market
Open and Current.

## Step 6: add a real timeout to the MRX call

Concurrency prevents one slow call from stopping the other workers, but a
Python thread cannot safely terminate a permanently stuck function. The real
timeout must be supplied by the MRX or network client.

Add it inside the real Market source function that the adapter calls. A minimal
example is:

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

Use the real timeout option supported by your MRX client:

- If it accepts seconds, use `timeout=5`.
- If it accepts milliseconds, use `timeout=5_000`.
- If it uses another name such as `request_timeout`, use that exact name.

Allow the timeout exception to leave the connector. `_load_market_frames()`
will catch it and retry the call four times.

Do not use this as a hard timeout:

```python
future.result(timeout=5)
```

That stops waiting for the result but does not stop the underlying stuck
thread. If the MRX client has no native timeout, its exact API must be checked
before adding a different process-based timeout.

## Step 7: update the integration-test imports

Open `tests/s07_integration.py`.

Add `Callable` and the two Barrier classes:

```python
from collections.abc import Callable
from datetime import datetime, timezone
from threading import Barrier, BrokenBarrierError, Event, Thread
```

Add `PRODUCT_SPECS` to the product import:

```python
from rebirth.domain.s02_products import PRODUCT_SPECS, ProductConnectorAdapter
```

## Step 8: add the small test manager

After the imports in `tests/s07_integration.py`, add:

```python
def _market_test_manager(
    loader: Callable[..., object],
    *,
    wait: Callable[[float], None] = lambda _seconds: None,
) -> RiskRefreshManager:
    return RiskRefreshManager(
        lambda _date: pd.DataFrame(),
        thresholds=lambda: pd.DataFrame(),
        risk_checker_loader=lambda _date: (pd.DataFrame(), pd.DataFrame()),
        market_status_resolver=lambda _date: "OFFICIAL",
        risk_loader=lambda _date, _source: pd.DataFrame(),
        market_open_loader=loader,
        market_status_loader=loader,
        sleep=wait,
    )
```

The injected `wait` function lets the retry test record delays without actually
sleeping.

## Step 9: add the concurrency and ordering test

Add this test to `tests/s07_integration.py`:

```python
def test_market_calls_overlap_and_keep_underlying_result_order() -> None:
    underlyings = ("CREDIT_A", "CREDIT_B", "CREDIT_C", "CREDIT_D")
    barrier = Barrier(len(underlyings))

    def loader(
        _source_type: str,
        _market_date: pd.Timestamp,
        underlying: str,
        *,
        market_status: str,
    ) -> pd.DataFrame:
        try:
            barrier.wait(timeout=2)
        except BrokenBarrierError as exc:
            raise ValueError("market calls did not overlap") from exc
        return pd.DataFrame(
            {"Underlying": [underlying], "Market Status": [market_status]}
        )

    manager = _market_test_manager(loader)
    spec = PRODUCT_SPECS["creditdelta"]
    opened = manager._load_product_market_open(
        spec,
        pd.Timestamp("2026-08-20"),
        underlyings,
        market_status="OFFICIAL",
    )
    current = manager._load_product_market_status(
        spec,
        pd.Timestamp("2026-08-21"),
        underlyings,
        market_status="OFFICIAL",
    )

    assert opened["Underlying"].tolist() == list(underlyings)
    assert current["Underlying"].tolist() == list(underlyings)
```

The Barrier can pass only when all four calls overlap. The final assertions
confirm that completion timing does not change returned order.

## Step 10: add the four-retry test

Add this test below the concurrency test:

```python
def test_market_call_retries_four_times_then_succeeds() -> None:
    calls = 0
    waits: list[float] = []

    def loader(
        _source_type: str,
        _market_date: pd.Timestamp,
        underlying: str,
        *,
        market_status: str,
    ) -> pd.DataFrame:
        nonlocal calls
        calls += 1
        if calls <= 4:
            raise ConnectionError("temporary market failure")
        return pd.DataFrame(
            {"Underlying": [underlying], "Market Status": [market_status]}
        )

    manager = _market_test_manager(loader, wait=waits.append)
    result = manager._load_product_market_status(
        PRODUCT_SPECS["creditdelta"],
        pd.Timestamp("2026-08-21"),
        ("CREDIT_A",),
        market_status="OFFICIAL",
    )

    assert calls == 5
    assert waits == [0.5, 0.5, 0.5, 0.5]
    assert result["Underlying"].tolist() == ["CREDIT_A"]
```

Five calls are expected: one initial attempt followed by four retries.

## Step 11: make existing call-order assertions concurrency-safe

Thread start order is not deterministic. Any test that compares the raw order
in which Open and Current connectors started must compare sorted calls instead.

Replace this pattern:

```python
assert [
    (source, underlying, status)
    for source, _, underlying, status in open_calls
] == [
    (source, underlying, status)
    for source, _, underlying, status in current_calls
]
```

With:

```python
assert sorted(
    (source, underlying, status)
    for source, _, underlying, status in open_calls
) == sorted(
    (source, underlying, status)
    for source, _, underlying, status in current_calls
)
```

Make the same change to the later assertion that uses
`open_calls[first_open_count:]` and `current_calls[first_open_count:]`.

This changes only connector start-order expectations. Production output order
remains stable because `_load_market_frames()` uses `executor.map()`.

## Step 12: run the tests

Run the focused integration tests first:

```powershell
python -m pytest tests/s07_integration.py -q
```

Then run the full suite:

```powershell
python -m pytest -q
```

Finally check the patch:

```powershell
git diff --check
git diff --stat
```

The focused tests must show no failures. The full suite must also pass before
publishing.

## Expected runtime behavior

For each product, the sequence is:

```text
validated Risk
  -> ordered unique Underlyings
  -> Open calls, maximum 20 at once
  -> concatenate Open frames in Underlying order
  -> Current calls, maximum 20 at once
  -> concatenate Current frames in Underlying order
  -> existing validation and P&L calculations
  -> atomic snapshot publication
```

If there are only four Underlyings, only four calls run. The remaining worker
capacity is unused.

If one call fails once, only that Underlying is retried. Successful Underlyings
are not deliberately repeated.

If one call still fails after four retries, the Market load raises an error and
the incomplete refresh is not published. The previous good snapshot remains
available.

## Common checks if it does not work

1. Confirm the real Market connector returns a pandas `DataFrame` for every
   Underlying, including empty results.
2. Confirm the connector accepts exactly one Underlying per call.
3. Confirm the timeout is inside the MRX/network call rather than around
   `future.result()`.
4. Confirm the service permits 20 concurrent calls. Reduce
   `_MARKET_MAX_WORKERS` if its connection limit is lower.
5. Confirm any shared MRX client or session is thread-safe. If it is not, use
   the client-supported session pattern or lower the worker count.
6. Read the warning log. It records source type, Underlying, retry number, and
   the connector error.
7. Treat `TypeError` and `ValueError` as schema or code faults; Fix 1 does not
   retry them.

## Rollback

To remove the complete code change from a repository that contains the original
Fix 1 commit:

```powershell
git revert 3d1ef5e
```

Do not manually delete only the thread pool while leaving the Open and Current
methods calling `_load_market_frames()`.
