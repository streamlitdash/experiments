# Fix 7: bulk-load FX Delta Market data

Use this fix when the FX Delta Market service can return every currency pair in
one request. It changes only FX Delta Market loading.

Before this change, `N` FX Delta underlyings cause:

```text
N Open calls + N Current calls = 2N source calls
```

After this change, they cause:

```text
1 bulk Open call + 1 bulk Current call = 2 source calls
```

Risk remains one separate call. FX Gamma, FX Vega, IR, Credit, Commodity and
all other products remain unchanged. Open and Current use different dates, so
keep them as two calls unless the upstream service genuinely returns both dates
in the same response.

## Files to change

In Rebirth V4.1 the files are:

```text
rebirth/services/s06_refresh.py
rebirth/adapters/s03_fx.py
the site-owned file that contains the real FX Delta Market API call
```

Your private package is named `cube` in the screenshots. If that is the package
you are editing, use the same relative paths under `cube/` instead of
`rebirth/`. Do not edit both copies.

Run these searches from the application root to find the live definitions:

```powershell
rg -n "def _load_product_market_open|def _load_product_market_status" cube rebirth
rg -n '"fx/delta"|FX_DELTA_OPEN|FX_DELTA_CURRENT|def get_delta_open|def get_delta_current' cube rebirth
```

The manager and adapter blocks below are paste-ready for the checked Rebirth
structure. The two source-call lines contain deliberate `YOUR_...` placeholders
because the private FX service name and its response-column names are not in
the repository. Replace those placeholders with the existing site call; do not
paste them literally.

## Step 1: add the FX Delta bulk branch for Open

Open `services/s06_refresh.py` in the live package.

Find `_load_product_market_open()`, then find this line inside it:

```python
selected_status = _require_market_status(market_status)
```

Immediately below that line, before `frames = []`, `def load_one(...)`, or any
thread-pool call, add:

```python
        if spec.source_type == "fx/delta" and adapter is not None:
            self._progress_activity(
                _callable_name(adapter.market_open),
                "market_open",
                source_type=spec.source_type,
                underlying="All FX Delta underlyings",
                product_index=1,
                product_total=1,
                message=f"Loading FX Delta Open for {len(underlyings)} underlyings.",
            )
            frame = adapter.market_open(
                open_date,
                underlyings,
                market_status=selected_status,
            )
            if not isinstance(frame, pd.DataFrame):
                raise TypeError("FX Delta bulk Open must return a pandas DataFrame")
            return frame.reset_index(drop=True)
```

Leave the existing per-underlying code immediately after this block. Every
other source type will continue through that existing path.

## Step 2: add the FX Delta bulk branch for Current

In the same file, find `_load_product_market_status()` and its line:

```python
selected_status = _require_market_status(market_status)
```

Immediately below it, before the existing per-underlying loader, add:

```python
        if spec.source_type == "fx/delta" and adapter is not None:
            self._progress_activity(
                _callable_name(adapter.market_status),
                "market_status",
                source_type=spec.source_type,
                underlying="All FX Delta underlyings",
                product_index=1,
                product_total=1,
                message=(
                    f"Loading FX Delta {selected_status} for "
                    f"{len(underlyings)} underlyings."
                ),
            )
            frame = adapter.market_status(
                market_date,
                underlyings,
                market_status=selected_status,
            )
            if not isinstance(frame, pd.DataFrame):
                raise TypeError("FX Delta bulk Current must return a pandas DataFrame")
            return frame.reset_index(drop=True)
```

Again, leave the existing loop or concurrent loader underneath it. Only
`fx/delta` returns early.

## Step 3: add a dedicated bulk FX Delta adapter

Open `adapters/s03_fx.py` in the live package.

Do not change the shared `_scalar_adapter()`. FX Gamma uses it too, so changing
its `underlying` argument to a tuple would change unrelated products.

Immediately after `_scalar_adapter()` and before `build_fx_adapters()`, add:

```python
def _bulk_delta_adapter(
    *,
    risk_source: RiskSource,
    open_source: MarketSource,
    current_source: MarketSource,
) -> ProductConnectorAdapter:
    """Bind FX Delta to one all-underlying call per Market leg."""

    def get_risk(risk_date: pd.Timestamp) -> pd.DataFrame:
        return exact_frame(
            risk_source(risk_date),
            columns=FX_DELTA_RISK,
            label="FX Delta risk",
        )

    def get_open(
        market_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame:
        return exact_frame(
            open_source(
                market_date,
                underlyings,
                market_status=market_status,
            ),
            columns=FX_DELTA_OPEN,
            label="FX Delta bulk Open",
        )

    def get_current(
        market_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame:
        frame = exact_frame(
            current_source(
                market_date,
                underlyings,
                market_status=market_status,
            ),
            columns=FX_DELTA_CURRENT,
            label="FX Delta bulk Current",
        )
        frame["Market Status"] = market_status
        return frame

    return ProductConnectorAdapter(
        risk=get_risk,
        market_open=get_open,
        market_status=get_current,
    )
```

The existing `MarketSource` and `ProductMarketConnector` annotations describe
ordinary single-underlying sources. This smallest fix deliberately passes a
tuple at runtime only for FX Delta. Python does not enforce those annotations;
all other adapters still receive one string. Step 8 below adjusts the one
existing integration assertion that assumes every recorded argument is a
string.

## Step 4: replace only the FX Delta registration

Still in `build_fx_adapters()`, find this existing block:

```python
        "fx/delta": _scalar_adapter(
            risk_source=delta_risk,
            open_source=delta_open,
            current_source=delta_current,
            risk_columns=FX_DELTA_RISK,
            open_columns=FX_DELTA_OPEN,
            current_columns=FX_DELTA_CURRENT,
            label="FX Delta",
        ),
```

Replace that entire block with:

```python
        "fx/delta": _bulk_delta_adapter(
            risk_source=delta_risk,
            open_source=delta_open,
            current_source=delta_current,
        ),
```

Do not replace the `fx/gamma` or `fx/vega` blocks.

## Step 5: change the real FX Delta Open source to one call

Find the real function supplied as `delta_open`. Its current shape will be
similar to:

```python
def get_delta_open(market_date, underlying, *, market_status):
    # one API request for one underlying
    ...
```

Replace that function with this shape:

```python
def get_delta_open(market_date, underlyings, *, market_status):
    # Replace only this call with the existing all-FX-Delta call. Preserve the
    # existing conversion from Live/OFFICIAL into the service's market label.
    raw = run_async(
        YOUR_ALL_FX_DELTA_OPEN_CALL(
            market_date,
            market_status=market_status,
        )
    )

    # The private RAMP helper shown in the existing code uses
    # DataFrame.from_dict(..., orient="index"). Reuse that conversion.
    frame = raw.copy() if isinstance(raw, pd.DataFrame) else get_ramp_read(raw)
    frame = frame.rename(
        columns={
            "YOUR_SOURCE_UNDERLYING_COLUMN": "Underlying",
            "YOUR_SOURCE_OPEN_COLUMN": "Open",
        }
    )
    frame["Underlying"] = frame["Underlying"].astype(str).str.strip()
    frame["Open"] = pd.to_numeric(frame["Open"], errors="raise")
    wanted = {str(value).strip() for value in underlyings}
    result = frame.loc[
        frame["Underlying"].isin(wanted),
        ["Underlying", "Open"],
    ].reset_index(drop=True)
    return result
```

Replace these three placeholders with the real names:

```text
YOUR_ALL_FX_DELTA_OPEN_CALL
YOUR_SOURCE_UNDERLYING_COLUMN
YOUR_SOURCE_OPEN_COLUMN
```

There must be no `for underlying in underlyings` loop around the API call. The
API is called once, then pandas filters its all-underlying response locally.

Keep the current Live/OFFICIAL routing logic. If the existing source maps FX
labels such as `EUR/USD` to `EURUSD`, keep that same normalization and apply it
to both `frame["Underlying"]` and `underlyings` before `.isin(wanted)`. Simple
whitespace stripping is sufficient only when the Risk and Market feeds already
use identical labels.

If the real API is synchronous, remove `run_async(...)` and use:

```python
raw = YOUR_ALL_FX_DELTA_OPEN_CALL(
    market_date,
    market_status=market_status,
)
```

## Step 6: change the real FX Delta Current source to one call

Find the function supplied as `delta_current` and replace it with this shape:

```python
def get_delta_current(market_date, underlyings, *, market_status):
    # Replace only this call with the existing all-FX-Delta call. Preserve the
    # existing conversion from Live/OFFICIAL into the service's market label.
    raw = run_async(
        YOUR_ALL_FX_DELTA_CURRENT_CALL(
            market_date,
            market_status=market_status,
        )
    )

    frame = raw.copy() if isinstance(raw, pd.DataFrame) else get_ramp_read(raw)
    frame = frame.rename(
        columns={
            "YOUR_SOURCE_UNDERLYING_COLUMN": "Underlying",
            "YOUR_SOURCE_CURRENT_COLUMN": "Current",
        }
    )
    frame["Underlying"] = frame["Underlying"].astype(str).str.strip()
    frame["Current"] = pd.to_numeric(frame["Current"], errors="raise")
    wanted = {str(value).strip() for value in underlyings}
    result = frame.loc[
        frame["Underlying"].isin(wanted),
        ["Underlying", "Current"],
    ].reset_index(drop=True)
    return result
```

Replace these placeholders:

```text
YOUR_ALL_FX_DELTA_CURRENT_CALL
YOUR_SOURCE_UNDERLYING_COLUMN
YOUR_SOURCE_CURRENT_COLUMN
```

If one service function supplies either date, the Open and Current functions
may call the same service function with different dates. They are still two
requests in total because `open_date` and `market_date` differ.

Apply the same existing FX label normalization described in Step 5 before the
Current `.isin(wanted)` filter.

## Step 7: optionally make the refresh call counter truthful

Back in `services/s06_refresh.py`, find:

```python
market_open_calls += len(requested_underlyings)
```

Replace it with:

```python
market_open_calls += (
    1 if source_type == "fx/delta" else len(requested_underlyings)
)
```

Then find:

```python
market_status_calls += len(requested_underlyings)
```

Replace it with:

```python
market_status_calls += (
    1 if source_type == "fx/delta" else len(requested_underlyings)
)
```

This changes only the displayed/logged call count. It does not affect P&L.
You may skip Step 7 if you want the smallest behavioral patch and do not use
this counter for diagnostics.

## Step 8: update the existing string-only test assertion

This is required only if the test suite contains the current integration
assertion that calls `.strip()` directly on every connector argument.

Open `tests/s07_integration.py` and find:

```python
assert all(call[2].strip() for call in open_calls)
```

Replace it with:

```python
assert all(
    all(str(value).strip() for value in call[2])
    if isinstance(call[2], tuple)
    else call[2].strip()
    for call in open_calls
)
```

The tuple occurs only for the new FX Delta bulk call. Existing single-
Underlying calls remain strings.

## Step 9: confirm the required output columns

The bulk Open source must return exactly:

```text
Underlying, Open
```

The bulk Current source must return exactly:

```text
Underlying, Current
```

The adapter attaches `Market Status` to Current after checking those exact
source columns. Do not return Portfolio, Group, Reported Underlying, Risk Type,
Risk Greek, or any tenor columns for FX Delta Market data.

Also confirm:

- `Underlying` is nonblank text;
- Open and Current are numeric and finite;
- each returned Underlying appears no more than once in each leg;
- only requested raw Underlyings are returned;
- missing quotes are omitted, not filled with zero.

The existing product validators will reject duplicates, invalid numerics, and
wrong columns before P&L is calculated.

## Step 10: run a simple check

First run a syntax check using the package name that you edited:

```powershell
python -m py_compile cube/services/s06_refresh.py cube/adapters/s03_fx.py
```

Or, for a `rebirth` package:

```powershell
python -m py_compile rebirth/services/s06_refresh.py rebirth/adapters/s03_fx.py
```

Then run a normal refresh. The progress display should show one FX Delta Open
activity and one FX Delta Current activity, each saying how many underlyings are
in the batch. It must not walk through individual FX Delta labels.

To inspect the result once while developing, temporarily add these lines just
before `return result` in each real source function:

```python
print(result.head())
print("FX Delta bulk rows:", len(result))
```

Remove temporary prints after confirming the shape.

## Expected runtime behavior

For FX Delta, the refresh now follows:

```text
one FX Delta Risk call
  -> derive the ordered raw Underlying scope
  -> one all-underlying Open API call for T-1
  -> filter the returned Open rows to that scope
  -> one all-underlying Current API call for Market Date
  -> filter the returned Current rows to that scope
  -> existing Open/Current validation
  -> existing P&L calculation
  -> existing atomic snapshot publication
```

This reduces request count and rate-limit pressure. It does not hold an extra
long-lived snapshot and it does not create a permanent module-level cache.

## Common mistakes

1. Adding the bulk branch after the old loop. It must appear before the loop or
   thread-pool loader so FX Delta returns early.
2. Changing `_scalar_adapter()`. That would also alter FX Gamma and may alter
   other scalar products.
3. Calling the all-underlying API inside a Python loop. That recreates the same
   rate-limit problem.
4. Returning every row from the source without filtering to the Risk-derived
   `underlyings` tuple.
5. Returning source column names instead of exactly `Underlying` plus `Open` or
   `Current`.
6. Adding Portfolio to Market data. Portfolio belongs to Risk; Market quotes
   remain unique at raw Underlying grain.
7. Filling a missing quote with zero. Missing Market data must stay missing.
8. Caching the bulk DataFrame permanently. A module-global cache can serve stale
   Live data on a same-date refresh.
9. Expecting one total request. Open and Current are two distinct legs and may
   use different dates.

## Rollback

To undo Fix 7 manually:

1. Remove the two early `if spec.source_type == "fx/delta"` blocks from
   `services/s06_refresh.py`.
2. Restore the two call counters to `len(requested_underlyings)`.
3. Restore the original `"fx/delta": _scalar_adapter(...)` registration.
4. Delete `_bulk_delta_adapter()`.
5. Restore the real Open and Current source functions to accept one
   `underlying: str` per call.

No stored data, saved views, Parquet history, or database schema is changed by
this fix.
