# Fix 6: concurrent Risk calls with retries

This guide implements concurrency, retries, and connector timeouts for Risk
calls only. It does not change Market Open or Market Current loading.

## Important scope

The current Risk contract is:

```python
risk(risk_date) -> pandas.DataFrame
```

One Risk product adapter is one task. Therefore:

- Ten separate Risk product adapters can create up to ten simultaneous calls.
- Four available Risk products create only four simultaneous calls.
- One Credit Risk adapter creates only one call, even when the worker limit is
  ten.

If a single Credit adapter internally has ten genuinely independent MRX
requests, those requests must be split inside that adapter. Do not call the same
adapter ten times without ten distinct request definitions, because that would
duplicate Risk rows.

## What Fix 6 changes

- Runs up to ten independent Risk product calls concurrently.
- Uses each product's own effective or forced Risk date.
- Retries a failed connector four times after the initial attempt.
- Waits 0.5 seconds between retry attempts.
- Keeps product results in the original `PRODUCT_SPECS` order.
- Validates every product sequentially after all connector calls return.
- Updates the staged `next_risk` dictionary only on the refresh thread.
- Leaves the existing atomic snapshot publication unchanged.

## Files to change

```text
rebirth/services/s06_refresh.py
tests/s07_integration.py
```

Run the commands in this guide from the Rebirth V4.1 repository root.

## Step 1: confirm the thread-pool import

Open `rebirth/services/s06_refresh.py`.

If you already implemented Fix 1, this import is present:

```python
from concurrent.futures import ThreadPoolExecutor
```

Otherwise, add it with the imports at the top of the file.
`ThreadPoolExecutor` is included with Python, so no package is required.

The existing `Mapping` and `Sequence` imports are also used by Fix 6:

```python
from typing import Callable, Mapping, Sequence
```

## Step 2: add the Risk settings

Find this line near the top of the file:

```python
LOGGER = logging.getLogger("rebirth.services.s06_refresh")
```

Add the Risk settings directly underneath it. If the Fix 1 Market settings are
already there, place these immediately after those settings instead:

```python
_RISK_MAX_WORKERS = 10
_RISK_RETRIES = 4
_RISK_RETRY_DELAY_SECONDS = 0.5
```

This means ten simultaneous Risk product calls and five possible attempts for a
failing product: one initial attempt plus four retries.

## Step 3: replace `_load_product_risk()`

Inside `RiskRefreshManager`, replace the complete existing
`_load_product_risk()` method with:

```python
    def _load_product_risk(
        self, spec: ProductSpec, risk_date: pd.Timestamp
    ) -> pd.DataFrame:
        adapter = self._connector_adapters.get(spec.source_type)
        connector = adapter.risk if adapter is not None else self._risk_loader
        connector_name = _callable_name(connector)
        self._progress_step(
            connector_name,
            "risk",
            source_type=spec.source_type,
            message="Loading connector risk.",
        )

        for retry in range(_RISK_RETRIES + 1):
            if retry > 0:
                self._progress_activity(
                    connector_name,
                    "risk",
                    source_type=spec.source_type,
                    message=(
                        f"Retrying Risk for {spec.source_type}: "
                        f"{retry} of {_RISK_RETRIES}."
                    ),
                )
            try:
                frame = (
                    adapter.risk(risk_date)
                    if adapter is not None
                    else self._risk_loader(risk_date, spec.source_type)
                )
            except (TypeError, ValueError) as exc:
                LOGGER.error(
                    "Risk connector failed for %s on %s: %s",
                    spec.source_type,
                    risk_date,
                    exc,
                )
                self._progress_activity(
                    connector_name,
                    "risk",
                    source_type=spec.source_type,
                    message="Risk connector failed.",
                )
                raise
            except Exception as exc:
                if retry == _RISK_RETRIES:
                    LOGGER.error(
                        "Risk connector exhausted retries for %s on %s: %s",
                        spec.source_type,
                        risk_date,
                        exc,
                    )
                    self._progress_activity(
                        connector_name,
                        "risk",
                        source_type=spec.source_type,
                        message="Risk connector failed after four retries.",
                    )
                    raise
                LOGGER.warning(
                    "Risk connector failed for %s; retry %d of %d: %s",
                    spec.source_type,
                    retry + 1,
                    _RISK_RETRIES,
                    exc,
                )
                self._sleep(_RISK_RETRY_DELAY_SECONDS)
                continue

            if not isinstance(frame, pd.DataFrame):
                LOGGER.error(
                    "Risk connector returned %s for %s on %s",
                    type(frame).__name__,
                    spec.source_type,
                    risk_date,
                )
                self._progress_activity(
                    connector_name,
                    "risk",
                    source_type=spec.source_type,
                    message="Risk connector returned an invalid result.",
                )
                raise TypeError("risk connector must return a pandas DataFrame")
            return frame

        raise AssertionError("unreachable risk retry state")
```

Only the connector call is retried. `TypeError`, `ValueError`, and later schema
validation failures are not retried.

The first call uses `_progress_step()` because it is one planned refresh work
unit. Retries use `_progress_activity()`, so retry attempts do not incorrectly
increase the planned progress total.

## Step 4: add `_load_risk_frames()`

Immediately after `_load_product_risk()`, add:

```python
    def _load_risk_frames(
        self,
        specs: Sequence[ProductSpec],
        risk_dates: Mapping[str, pd.Timestamp],
    ) -> list[pd.DataFrame]:
        """Load independent Risk products concurrently in catalogue order."""

        def load_one(spec: ProductSpec) -> pd.DataFrame:
            risk_date = risk_dates[spec.source_type]
            return self._load_product_risk(spec, risk_date)

        with ThreadPoolExecutor(max_workers=_RISK_MAX_WORKERS) as executor:
            return list(executor.map(load_one, specs))
```

`executor.map()` is important. Calls can finish in any order, but returned
frames remain aligned with the original `specs` sequence.

The correct Risk date is resolved inside each task using
`risk_dates[spec.source_type]`. This preserves checker Age, forced dates, and
historical view dates.

## Step 5: replace only the sequential Risk connector loop

Still in `rebirth/services/s06_refresh.py`, find the refresh section that starts
with:

```python
risk_specs = [
    spec
    for spec in PRODUCT_SPECS.values()
    if spec.source_type in changed_source_types
]
```

Keep the existing `risk_product_delay` calculation. Replace only the following
sequential `for` loop with this block:

```python
                raw_risk_frames = self._load_risk_frames(risk_specs, next_dates)

                for product_index, (spec, raw_risk) in enumerate(
                    zip(risk_specs, raw_risk_frames, strict=True),
                    start=1,
                ):
                    source_type = spec.source_type
                    risk_date = next_dates[source_type]
                    product_label = _product_progress_label(spec)
                    self._progress_step(
                        f"get_{spec.key}_risk",
                        "risk",
                        source_type=source_type,
                        product_label=product_label,
                        product_index=product_index,
                        product_total=len(risk_specs),
                        hold_seconds=risk_product_delay,
                        message=(
                            f"Loading and validating Risk/dRisk for "
                            f"{product_label}."
                        ),
                    )
                    next_risk[source_type] = get_product_risk(
                        spec,
                        risk_date,
                        raw_risk,
                    )
                    if risk_product_delay > 0:
                        self._sleep(risk_product_delay)
```

Do not move `get_product_risk()` into the retry loop. Invalid schemas should
fail immediately rather than being downloaded five times.

The validation and `next_risk` assignments remain sequential and deterministic.
The existing planned progress count of two units per changed Risk product also
remains correct:

1. One connector-start progress step.
2. One validation progress step.

## Step 6: add the timeout to each real Risk connector

The pool cannot safely kill a permanently stuck Python thread. Put the timeout
on the real MRX request inside the Risk adapter.

Example:

```python
RISK_TIMEOUT_SECONDS = 5


def get_credit_risk(risk_date):
    try:
        return mrx_risk_call(
            risk_date,
            timeout=RISK_TIMEOUT_SECONDS,
        )
    except TimeoutError as exc:
        raise TimeoutError(
            f"Credit Risk call timed out for {risk_date:%Y-%m-%d}"
        ) from exc
```

Use the real option accepted by the MRX client:

- `timeout=5` when it expects seconds.
- `timeout=5_000` when it expects milliseconds.
- The client's exact name, such as `request_timeout`, when different.

Allow the timeout exception to propagate. `_load_product_risk()` will retry it.

Do not use `future.result(timeout=5)` as a hard timeout. It stops waiting but
does not terminate the underlying call.

## Step 7: understand the single-Credit limitation

This manager-level fix parallelizes product adapters. For example, Credit
Delta, Credit Vega, FX Delta, and IR Delta can load together when all four are
part of `changed_source_types`.

If only Credit Delta is loaded, `risk_specs` contains one item and the executor
starts one task. Ten configured workers do not create ten copies of that task.

To obtain ten calls from one Credit adapter, first define ten non-overlapping
MRX requests inside that adapter. Each request must own a distinct, real slice
of Risk data. Concatenate the ten results once and return one frame from the
adapter.

Do not invent slices by repeating the same query. Do not remove duplicate Risk
rows afterward to hide repeated calls.

Also avoid enabling a ten-worker manager pool and a ten-worker pool inside every
adapter without calculating the total. That could create up to 100 concurrent
MRX calls.

## Step 8: add the Risk test manager

Open `tests/s07_integration.py`.

Fix 1 already imports `Callable`, `Barrier`, `BrokenBarrierError`, and
`PRODUCT_SPECS`. If they are absent, the relevant imports should be:

```python
from collections.abc import Callable
from threading import Barrier, BrokenBarrierError, Event, Thread

from rebirth.domain.s02_products import PRODUCT_SPECS, ProductConnectorAdapter
```

Add this helper near `_market_test_manager()`:

```python
def _risk_test_manager(
    loader: Callable[..., object],
    *,
    wait: Callable[[float], None] = lambda _seconds: None,
) -> RiskRefreshManager:
    return RiskRefreshManager(
        lambda _date: pd.DataFrame(),
        thresholds=lambda: pd.DataFrame(),
        risk_checker_loader=lambda _date: (pd.DataFrame(), pd.DataFrame()),
        market_status_resolver=lambda _date: "OFFICIAL",
        risk_loader=loader,
        market_open_loader=lambda *_args, **_kwargs: pd.DataFrame(),
        market_status_loader=lambda *_args, **_kwargs: pd.DataFrame(),
        sleep=wait,
    )
```

## Step 9: add the Risk concurrency and order test

Add:

```python
def test_risk_calls_overlap_and_keep_product_result_order() -> None:
    specs = (
        PRODUCT_SPECS["creditdelta"],
        PRODUCT_SPECS["fxdelta"],
        PRODUCT_SPECS["irdelta"],
        PRODUCT_SPECS["commodelta"],
    )
    barrier = Barrier(len(specs))
    risk_dates = {
        specs[0].source_type: pd.Timestamp("2026-08-20"),
        specs[1].source_type: pd.Timestamp("2026-08-19"),
        specs[2].source_type: pd.Timestamp("2026-08-18"),
        specs[3].source_type: pd.Timestamp("2026-08-17"),
    }

    def loader(
        risk_date: pd.Timestamp,
        source_type: str,
    ) -> pd.DataFrame:
        try:
            barrier.wait(timeout=2)
        except BrokenBarrierError as exc:
            raise ValueError("Risk calls did not overlap") from exc
        return pd.DataFrame(
            {
                "Source Type": [source_type],
                "Risk Date": [pd.Timestamp(risk_date)],
            }
        )

    manager = _risk_test_manager(loader)
    frames = manager._load_risk_frames(specs, risk_dates)

    assert [frame.loc[0, "Source Type"] for frame in frames] == [
        spec.source_type for spec in specs
    ]
    assert [frame.loc[0, "Risk Date"] for frame in frames] == [
        risk_dates[spec.source_type] for spec in specs
    ]
```

The Barrier proves that the four connector calls overlap. The assertions prove
that results still match catalogue order and receive the correct date.

## Step 10: add the Risk retry test

Add:

```python
def test_risk_call_retries_four_times_then_succeeds() -> None:
    calls = 0
    waits: list[float] = []

    def loader(
        risk_date: pd.Timestamp,
        source_type: str,
    ) -> pd.DataFrame:
        nonlocal calls
        calls += 1
        if calls <= 4:
            raise ConnectionError("temporary Risk failure")
        return pd.DataFrame(
            {
                "Source Type": [source_type],
                "Risk Date": [pd.Timestamp(risk_date)],
            }
        )

    spec = PRODUCT_SPECS["creditdelta"]
    risk_dates = {spec.source_type: pd.Timestamp("2026-08-20")}
    manager = _risk_test_manager(loader, wait=waits.append)
    frames = manager._load_risk_frames((spec,), risk_dates)

    assert calls == 5
    assert waits == [0.5, 0.5, 0.5, 0.5]
    assert frames[0].loc[0, "Source Type"] == spec.source_type
```

Five total calls confirm one initial attempt and four retries.

Add one final test to confirm deterministic connector failures are not retried:

```python
def test_risk_value_error_is_not_retried() -> None:
    calls = 0
    waits: list[float] = []

    def loader(
        _risk_date: pd.Timestamp,
        _source_type: str,
    ) -> pd.DataFrame:
        nonlocal calls
        calls += 1
        raise ValueError("invalid Risk schema")

    spec = PRODUCT_SPECS["creditdelta"]
    risk_dates = {spec.source_type: pd.Timestamp("2026-08-20")}
    manager = _risk_test_manager(loader, wait=waits.append)

    with pytest.raises(ValueError, match="invalid Risk schema"):
        manager._load_risk_frames((spec,), risk_dates)

    assert calls == 1
    assert waits == []
```

## Step 11: run the checks

Run the focused integration tests:

```powershell
python -m pytest tests/s07_integration.py -q
```

Then run the complete test suite:

```powershell
python -m pytest -q
```

Finally inspect the patch:

```powershell
git diff --check
git diff --stat
```

Do not publish until both test commands complete without failures.

## Expected flow

```text
checker/readiness
  -> changed Risk product specifications
  -> resolve each source's effective Risk date
  -> up to 10 adapter calls concurrently
  -> ordered raw Risk frames
  -> sequential get_product_risk validation
  -> staged next_risk dictionary
  -> existing Market and P&L work
  -> one atomic snapshot publication
```

If a Risk connector ultimately fails, the refresh does not reach snapshot
publication. The application retains the last good committed snapshot.

This smallest implementation temporarily holds the returned raw Risk frames in
a list until sequential validation finishes. If those frames are very large,
watch process memory and reduce `_RISK_MAX_WORKERS`. A streaming result design
is possible, but it is a separate optimization and adds more coordination code.

## Troubleshooting

1. Confirm there is more than one item in `risk_specs`; otherwise concurrency
   cannot improve the manager-level Risk load.
2. Confirm every adapter uses the `risk_date` passed to it rather than choosing
   another date internally.
3. Confirm the real MRX timeout is inside the connector.
4. Confirm the MRX client and any shared session are thread-safe.
5. Reduce `_RISK_MAX_WORKERS` if the service permits fewer than ten concurrent
   calls.
6. Treat `TypeError` and `ValueError` as code or schema failures rather than
   transient failures.
7. Use warning and error logs to identify the actual failed Source Type and
   Risk date. Concurrent progress text can show whichever Risk worker updated
   most recently, so the log is the reliable failure record.
8. Never concatenate separate products into one frame. Each product is still
   validated independently against its own `ProductSpec`.

## Rollback

Revert the commit that applies Fix 6, or reverse these four changes together:

1. Restore the original sequential Risk loop.
2. Restore the original `_load_product_risk()` method.
3. Remove `_load_risk_frames()`.
4. Remove the three `_RISK_*` constants.

Do not remove only `_load_risk_frames()` while leaving the refresh loop calling
it.
