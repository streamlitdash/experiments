# Fix 10 — Modular cold-start simplification after Fix 4

Use this guide only after **Fix 4 has been implemented completely**. Fix 4
already removes Portfolio from the Risk analytical layer, collapses the
published Risk dashboard across Portfolio, keeps detailed Portfolio data in
`combined_pl`, and preserves Portfolio filters on Stock and P&L.

This guide removes or defers the remaining duplicate and defensive work around
cold startup. It does not change connector values, market identity, P&L
formulas, Portfolio mapping, Reported Underlying mapping, or atomic publishing.

Every numbered chapter is an independent patch. For example, Steps 2, 5, and 6
may be implemented without Steps 1, 3, 4, 7, or 8. Each step has its own test
and rollback boundary. Do not combine several steps in one commit until each
one has passed independently with the real Credit-sized data.

## 0. Required baseline and independence rules

Confirm Fix 4 first:

```python
snapshot = manager.snapshot

assert "Portfolio" in snapshot.combined_pl
assert "Portfolio" not in snapshot.dashboard_frame
```

The following must remain true throughout Fix 10:

1. Raw Risk stays at Portfolio grain.
2. Open and Current remain unique at quote grain.
3. Risk joins Market `many_to_one`.
4. P&L is calculated before Portfolio is collapsed.
5. Portfolio and Reported Underlying mappings remain authoritative.
6. Missing Market data remains unavailable; it is not changed to zero.
7. Atomic commit and last-good snapshot retention remain enabled.
8. RiskChecker `Age` remains available for Risk-date selection.

### Independent step matrix

| Step | Change | Expected cold-start effect | Requires another Fix 10 step |
|---|---|---:|---|
| 1 | Prevent concurrent heavy materialisation after true completion | High | None |
| 2 | Remove two unnecessary P&L input copies | High with detailed Credit Risk | None |
| 3 | Use one owned validation frame instead of a copy chain | High | None |
| 4 | Move the full position sort out of P&L calculation | Medium to high | None |
| 5 | Reuse one combined MarketBook | Medium | None |
| 6 | Build SearchCatalog on first use | Medium after Fix 4 | None |
| 7 | Use one owned dashboard copy during UI preparation | Low to medium after Fix 4 | None |
| 8 | Defer initial Risk tables to their existing callbacks | Low to medium after Fix 4 | None |

Steps 2 and 3 both edit `rebirth/domain/s03_calculations.py`, but they edit
different ownership boundaries and do not depend on one another. Steps 5 and 6
both involve SearchCatalog, but each chapter explicitly preserves the old API
needed when the other step is absent.

Before every step, record a baseline using the same live dates:

```text
cold-start total seconds
peak process RSS
Risk rows loaded
mapped combined_pl rows
dashboard_frame rows
Market rows
time at combining/validation/publish
time at synchronising browser
```

Do not judge a change from the browser's reconnecting message alone. Confirm
whether the Python process exited, was killed for memory, or merely stopped
answering HTTP while a CPU-heavy callback ran.

---

## Step 1 — Gate heavy materialisation on true worker completion

### Purpose

Prevent several 500 ms interval callbacks from concurrently copying, preparing,
and rendering the completed dashboard while the refresh worker still holds its
final local frames.

This step does not alter the refresh pipeline or any financial frame.

### Files

- `rebirth/app/s04_startup.py`
- `rebirth/pages/risk/s15_refresh.py`
- `rebirth/ui/s04_components.py`
- `tests/s12_startup.py`
- `tests/s16_refreshshell.py`

### 1. Stop every coordinator entry point reporting success too early

The revision shortcut exists in `start()`, `schedule_start()`, and `status()`.
Change all three. Otherwise the next interval calls `start()` and marks startup
successful before the corrected `status()` method is reached.

In each method, calculate worker liveness before promoting a published revision
to success:

Use this rule:

```python
worker_alive = self._worker is not None and self._worker.is_alive()

if self._revision() > 0:
    if not worker_alive:
        self._phase = "succeeded"
        self._error = None
    return False  # In start()/schedule_start(); status() continues below.
```

`StartupCoordinator._run()` remains the normal owner of success. It calls
`_succeed()` only after `refresh()` has returned and its large local variables
can begin to leave scope.

Do not change `copy_result=False` in `_run()`.

In `status()`, use the same condition without the `return False`:

```python
worker_alive = self._worker is not None and self._worker.is_alive()
if self._revision() > 0 and not worker_alive:
    self._phase = "succeeded"
    self._error = None
```

Keep the existing elapsed-time/watchdog calculation after it.

### 2. Make the external-writer follower wait for true completion

`_follow_existing_writer()` currently succeeds as soon as a revision appears.
Read the manager's progress first and require both a revision and a stopped
writer:

```python
def _follow_existing_writer(self, attempt: int) -> None:
    while True:
        try:
            progress = self._manager.progress
            running = bool(progress.running)
            progress_error = progress.error
        except Exception as error:
            self._fail(attempt, error)
            return

        if self._revision() > 0 and not running:
            self._succeed(attempt)
            return
        if not running:
            self._fail(
                attempt,
                RuntimeError(
                    str(progress_error)
                    if progress_error
                    else "The existing refresh ended without publishing revision 1."
                ),
            )
            return
        sleep(0.2)
```

### 3. Add one non-blocking materialisation lock

In `register_refresh_callbacks()` in `rebirth/pages/risk/s15_refresh.py`, create one
process-owned lock next to the existing coordinator:

```python
from threading import Lock

startup_hydration_lock = Lock()
```

In `load_initial_snapshot_after_first_paint()`, acquire it only around the
heavy materialisation:

```python
if startup.phase == "succeeded" and refresh_manager.health.revision > 0:
    if not startup_hydration_lock.acquire(blocking=False):
        raise PreventUpdate
    try:
        return materialize_initial_dashboard(
            refresh_manager.control_snapshot
        )
    finally:
        startup_hydration_lock.release()
```

The interval may continue to poll lightweight status while another Python
materialisation is active, but it cannot create a second concurrent dashboard
preparation in the same process. A later interval or second browser may prepare
the dashboard after the first callback releases the lock. This is a
**single-flight concurrency cap**, not a guarantee that the function executes
only once for the lifetime of the process.

The lock covers Python materialisation until the callback returns. Dash may
still serialize and send the returned component tree after the `finally` block,
so do not claim that it protects the entire HTTP response lifetime.

### 4. Slow only the lightweight cold-page poll

Change `initial-load-trigger` from 500 ms to 1,000 ms:

```python
dcc.Interval(
    id="initial-load-trigger",
    interval=1_000,
    n_intervals=0,
    max_intervals=-1,
    disabled=error is not None and not keep_polling,
)
```

Do not remove `/healthz`, `/progressz`, the refresh lock, or the atomic commit.
Their payloads are small and they are useful for diagnosing a dead process.
Changing 500 ms to 1,000 ms only halves polling traffic; the lock and completion
gate above provide the actual concurrency protection.

### Tests

Add tests proving:

1. A fake refresh blocked after publishing revision 1 does not report
   `succeeded`; `running` or watchdog `stalled` are both valid.
2. Exercise that assertion through `start()`, `schedule_start()`, and
   `status()`.
3. A followed external writer does not succeed until `progress.running` is
   false.
4. Success is reported after the owned worker returns.
5. Two simultaneous materialisation attempts call
   `materialize_initial_dashboard()` at most once concurrently.
6. A skipped browser can materialise on a later poll.
7. Refresh failure still exposes Retry and retains the last good revision.

Run:

```powershell
python -m pytest tests/s12_startup.py -q
python -m pytest tests/s16_refreshshell.py -q
```

### Rollback

Revert only the coordinator success condition, hydration lock, and interval
value. No snapshot or saved data migration is involved.

### Independence

This step uses the existing snapshot, SearchCatalog, MarketBook, validators,
and UI preparation unchanged. It does not require any other Fix 10 step.

---

## Step 2 — Remove the two unnecessary copies in `get_product_pl()`

### Purpose

Avoid copying the complete validated Risk and Market frames immediately before
`pandas.merge()` creates a new result.

This is the smallest high-value change in this guide.

### Files

- `rebirth/domain/s03_calculations.py`
- `tests/s04_market.py`

### Replace the Risk ownership branch

Replace:

```python
if validated_risk is not None:
    if risk_source is not None:
        raise ValueError(
            "validated_risk cannot be combined with a raw risk source"
        )
    risk = validated_risk.copy()
```

with:

```python
if validated_risk is not None:
    if risk_source is not None:
        raise ValueError(
            "validated_risk cannot be combined with a raw risk source"
        )
    risk = validated_risk
```

Replace:

```python
if validated_market is not None:
    if open_source is not None or status_source is not None:
        raise ValueError(
            "validated_market cannot be combined with raw market sources"
        )
    market = validated_market.copy()
```

with:

```python
if validated_market is not None:
    if open_source is not None or status_source is not None:
        raise ValueError(
            "validated_market cannot be combined with raw market sources"
        )
    market = validated_market
```

Do not make any in-place assignments to `risk` or `market` after this change.
All P&L assignments must remain on `result`, which is created by:

```python
result = risk.merge(
    market,
    on=spec.market_keys,
    how="left",
    validate="many_to_one",
    indicator="_market_merge",
)
```

Do not remove `validate="many_to_one"`.

### Tests

Add a test that deep-copies the two inputs, calls `get_product_pl()`, and proves
both inputs are unchanged:

```python
expected_risk = risk.copy(deep=True)
expected_market = market.copy(deep=True)

result = get_product_pl(
    spec,
    risk_date,
    validated_risk=risk,
    validated_market=market,
    market_date=market_date,
    market_status="OFFICIAL",
)

pd.testing.assert_frame_equal(risk, expected_risk)
pd.testing.assert_frame_equal(market, expected_market)
assert result is not risk
assert result is not market
```

Run:

```powershell
python -m pytest tests/s04_market.py -q
python -m pytest tests/s07_integration.py -q
```

### Rollback

Restore `.copy()` on those two assignments.

### Independence

This step does not change validation, sorting, dashboard aggregation, Market
combination, SearchCatalog, or browser hydration.

---

## Step 3 — Validate one owned frame instead of repeatedly copying it

### Purpose

Keep every connector and financial validation while avoiding a succession of
full-frame copies in `_load_frame()`, `_enforce_product()`,
`_require_nonblank()`, `_coerce_numeric()`, tenor-order validation, and final
projection.

This step is deliberately separate from Step 2. Step 2 concerns already
validated P&L inputs; this step concerns connector validation.

### Files

- `rebirth/domain/s03_calculations.py`
- `rebirth/domain/s07_governance.py`
- `tests/s03_adapters.py`
- `tests/s04_market.py`
- `tests/s07_integration.py`
- `tests/s14_reporting.py`

### 1. Keep exactly one ownership copy in `_load_frame()`

Leave this boundary unchanged:

```python
return frame.copy()
```

That one copy protects a connector-owned or cached source frame from mutation.
It becomes the owned working frame for the remaining validator chain.

### 2. Give shared mutation helpers an explicit ownership flag

`_require_nonblank()` and `_coerce_numeric()` are imported by governance code,
including public `merge_config()`. They must remain defensive by default. Add:

```python
def _require_nonblank(
    frame: pd.DataFrame,
    columns: list[str],
    label: str,
    *,
    owned: bool = False,
) -> pd.DataFrame:
    result = frame if owned else frame.copy()
    # Keep the existing validation loop.
```

```python
def _coerce_numeric(
    frame: pd.DataFrame,
    columns: list[str],
    label: str,
    *,
    owned: bool = False,
) -> pd.DataFrame:
    result = frame if owned else frame.copy()
    # Keep the existing validation loop.
```

Pass `owned=True` only from the Risk/Open/Current chains after `_load_frame()`
has created the private working copy:

```python
frame = _require_nonblank(
    frame,
    key_columns,
    f"{spec.key} risk",
    owned=True,
)
frame = _coerce_numeric(
    frame,
    [RISK, DRISK, VOL_SCORE],
    f"{spec.key} risk",
    owned=True,
)
```

Use the same `owned=True` pattern for optional Credit measures and the
Open/Current validation chains.

Leave governance and service calls unchanged. Their omitted flag keeps
`owned=False`, so public `merge_config()` cannot mutate its supplied P&L frame.

### 3. Let pipeline-only validators operate on their owned input

In these two private helpers, replace `result = frame.copy()` with
`result = frame`:

- `_enforce_product()`
- `_validate_market_tenor_orders()`

Their current pipeline inputs come from `_load_frame()` or from an internally
created merged Market frame. Do not export or reuse them as general
caller-owned mutation helpers.

These are private pipeline helpers. They receive the frame returned by
`_load_frame()` in normal Risk/Open/Current validation.

Do not remove any of their checks. In particular, retain:

- boolean rejection in numeric columns;
- non-finite rejection;
- null/blank key rejection;
- Risk Type and Risk Greek enforcement;
- one tenor-to-order and one order-to-tenor authority;
- duplicate position and quote-key checks.

### 4. Avoid the final projection copy when the working frame is already owned

At the end of `get_product_risk()` replace:

```python
return frame[columns].copy()
```

with:

```python
return frame.loc[:, columns]
```

Use the same rule only in Open/Current validators where the selected frame is
already the one `_load_frame()` owns. Do not apply this mechanically to public
helpers that may return a view into a caller-owned DataFrame.

### 5. Add source-immutability tests before changing further copies

For Risk, Open, and Current:

1. Construct the connector DataFrame.
2. Keep a deep expected copy.
3. Run the public validator.
4. Prove the connector frame is unchanged.
5. Prove the returned frame has canonical dtypes and columns.

This test is the deletion gate. If it fails, restore the copy at the actual
ownership boundary rather than adding copies back to every helper.

Also deep-copy a P&L frame before `merge_config()` and assert that the public
call leaves it unchanged. This specifically protects the governance caller
that uses `_require_nonblank()` with the defensive default.

### Tests

Run:

```powershell
python -m pytest tests/s03_adapters.py -q
python -m pytest tests/s04_market.py -q
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s14_reporting.py -q
```

Then test the real Credit connector and log peak RSS before and after
`get_product_risk()`.

### Rollback

Restore `result = frame.copy()` one helper at a time. The source-immutability
tests identify the first boundary that genuinely needs ownership.

### Independence

This step does not rely on Step 2. If Step 2 is absent, `get_product_pl()` may
still copy its validated inputs. If Step 2 is present, both savings compose.

---

## Step 4 — Move full position sorting out of P&L calculation

### Purpose

Avoid allocating large sort index arrays for every Portfolio-grain P&L frame.
This step deliberately retires the current incidental output-row-order
contract: `get_product_pl()` will return merge/Risk-input order, while values
remain keyed by the full position identity. Apply this step only if callers do
not use row position as identity.

### Files

- `rebirth/domain/s03_calculations.py`
- `tests/s04_market.py`

### 1. Remove only the ordinary full-frame sort

In `get_product_pl()`, replace:

```python
sort_columns = [UNDERLYING, *spec.tenor_order_columns, PORTFOLIO]
result = result.sort_values(
    sort_columns,
    kind="stable",
    na_position="last",
).reset_index(drop=True)
```

with:

```python
result = result.reset_index(drop=True)
```

Do not change the market-key merge or P&L formula.

Keep the Gamma split ordering initially. Gamma creates sourced and derived rows
and its existing local order makes those paired rows easier to inspect. It can
be considered separately after this step is proven.

### 2. Sort only at a true presentation boundary

No UI change should be needed initially. The current presentation boundaries
already apply connector-owned tenor ranks in Quick Risk/Quick Market
(`rebirth/domain/s10_search.py`), Risk hierarchy/detail tables
(`rebirth/ui/s02_aggregation.py`), and P&L-send output
(`rebirth/domain/s08_pnl.py`). Verify those consumers rather than adding a
second global sort.

Do not add Portfolio back to the Fix 4 Risk dashboard merely to reproduce the
old position order.

The current Risk archive validator guarantees content and schema, not Parquet
row order. Therefore this step makes no archive change. If byte-reproducible
archive row order later becomes a requirement, implement that as a separate
archive-boundary change rather than coupling it to this step.

### Tests

Financial tests must compare keyed values, not incidental input row order:

```python
keys = [
    column
    for column in (
        SOURCE_TYPE,
        RISK_TYPE,
        RISK_GREEK,
        SPLIT,
        UNDERLYING,
        *spec.tenor_columns,
        PORTFOLIO,
        REGION,
    )
    if column in actual.columns
]
actual_keyed = actual.sort_values(
    keys,
    kind="stable",
    na_position="last",
).reset_index(drop=True)
expected_keyed = expected.sort_values(
    keys,
    kind="stable",
    na_position="last",
).reset_index(drop=True)
pd.testing.assert_frame_equal(actual_keyed, expected_keyed)
```

The existing join test in `tests/s04_market.py` asserts a returned row order.
Rewrite it to compare by the complete identity above. This formally retires
position-row order as a calculation contract while keeping tenor order as
presentation authority.

Also keep one focused Gamma assertion that every position's sourced Risk row
is still before its derived Gamma row; this step intentionally leaves the
Gamma-local ordering block in place.

Run:

```powershell
python -m pytest tests/s04_market.py -q
python -m pytest tests/s07_integration.py -q
```

### Rollback

Restore the `sort_values()` block. No stored schema changes.

### Independence

This step does not change copies, SearchCatalog, MarketBook, dashboard grain,
or UI hydration.

---

## Step 5 — Reuse one combined MarketBook

### Purpose

Stop retaining one combined MarketBook for the snapshot and constructing or
deep-copying another equivalent MarketBook for Quick Market/SearchCatalog.

This chapter deliberately keeps `snapshot.market_frame`, so Data and Quick
Market APIs remain available and Step 6 is not required.

### Files

- `rebirth/services/s02_state.py`
- `rebirth/services/s06_refresh.py`
- `rebirth/domain/s10_search.py`
- `tests/s07_integration.py`
- `tests/s10_reads.py`

### 1. Build `combined_market` once

Keep the existing call in `RiskRefreshManager.refresh()`:

```python
combined_market = self._combined_market_frame(
    next_market,
    market_date,
)
```

Continue assigning that exact frame to:

```python
snapshot.market_frame
```

### 2. Let SearchCatalog accept the already combined frame

Add an optional `market_frame` argument to `build_search_catalog()` while
retaining the old `market_frames` argument for compatibility:

```python
def build_search_catalog(
    *,
    revision: int,
    risk_frames: Mapping[str, pd.DataFrame],
    risk_pivot_frame: pd.DataFrame | None = None,
    market_frame: pd.DataFrame | None = None,
    market_frames: Mapping[str, pd.DataFrame] | None = None,
    # existing remaining arguments
) -> SearchCatalog:
    if market_frame is None:
        if market_frames is None:
            raise ValueError(
                "market_frame or market_frames is required"
            )
        owned_market_frame = _market_catalog_frame(
            market_frames,
            market_date,
        )
    else:
        owned_market_frame = _market_catalog_from_combined(
            market_frame,
            market_date,
        )
```

Existing external/test callers can continue supplying `market_frames`.

The snapshot MarketBook may contain more columns than SearchCatalog's narrow
`MARKET_RESULT_COLUMNS` contract. Add a small projection helper rather than
concatenating every product again:

```python
def _market_catalog_from_combined(
    market_frame: pd.DataFrame,
    market_date: pd.Timestamp,
) -> pd.DataFrame:
    result = market_frame.loc[:, list(MARKET_RESULT_COLUMNS)].copy()
    expected_date = pd.Timestamp(market_date).normalize()
    supplied_dates = pd.to_datetime(
        result[MARKET_DATE],
        errors="coerce",
    ).dt.normalize()
    invalid = supplied_dates.isna() | supplied_dates.ne(expected_date)
    if invalid.any():
        rows = result.index[invalid].tolist()[:5]
        raise ValueError(
            "combined MarketBook contains a missing or stale Market Date "
            f"at rows {rows}"
        )
    result[MARKET_DATE] = supplied_dates
    return result.sort_values(
        [
            SOURCE_TYPE,
            UNDERLYING,
            *TENOR_ORDER_COLUMNS,
            *TENOR_COLUMNS,
        ],
        kind="stable",
        na_position="last",
    ).reset_index(drop=True)
```

When `market_frame` is supplied, call this projection helper. When only
`market_frames` is supplied, retain the existing `_market_catalog_frame()`
compatibility path.

### 3. Use the prebuilt frame in the manager

Change `_build_snapshot_search_catalog()` to accept optional `market_frame` and
forward it to `build_search_catalog()` while continuing to pass the existing
`dashboard` argument as `risk_pivot_frame=dashboard`.

The normal financial refresh passes:

```python
search_catalog = self._build_snapshot_search_catalog(
    revision=revision,
    risk_frames=next_risk,
    market_frames=next_market,
    market_frame=combined_market,
    dashboard=dashboard,
    risk_dates=next_dates,
    market_date=market_date,
    market_status=expected_market_status,
)
```

The portfolio-only refresh is a separate eager SearchCatalog path. It does not
rebuild Market, so pass its existing committed frame:

```python
search_catalog = self._build_snapshot_search_catalog(
    revision=revision,
    risk_frames=risk_frames,
    market_frames=market_frames,
    market_frame=base_snapshot.market_frame,
    dashboard=dashboard,
    risk_dates=risk_dates,
    market_date=market_date,
    market_status=base_snapshot.market_status,
)
```

The compatibility `market_frames` argument remains available, but both
manager-owned paths prefer `market_frame`. Do not rename the public
`risk_pivot_frame` parameter to `dashboard`.

### 4. Remove the defensive SearchCatalog MarketBook deep copy only for the manager path

Give `SearchCatalog.__init__()` an ownership flag whose public default remains
defensive:

```python
def __init__(
    self,
    *,
    market_frame: pd.DataFrame,
    copy_market_frame: bool = True,
    # existing arguments
):
    self._market_frame = (
        market_frame.copy(deep=True)
        if copy_market_frame
        else market_frame
    )
```

Both `_market_catalog_frame()` and `_market_catalog_from_combined()` return an
owned, narrow catalog projection. Therefore `build_search_catalog()` passes:

```python
return SearchCatalog(
    revision=revision,
    risk_dates=risk_dates,
    market_date=market_date,
    market_frame=owned_market_frame,
    risk_pivot_frame=risk_pivot_frame,
    copy_market_frame=False,
)
```

Direct public/test construction of `SearchCatalog(...)` keeps the defensive
default `copy_market_frame=True`.

Search methods must remain read-only. Never assign into
`self._market_frame`.

### Tests

Add tests proving:

1. `_combined_market_frame()` is called at most once per financial refresh and
   zero times for a portfolio-only refresh.
2. Quick Market totals and rows are unchanged.
3. Search methods do not mutate `snapshot.market_frame`.
4. A public `SearchCatalog` constructed with default arguments still protects
   its input with a copy.
5. A mixed, missing, or stale Market Date in the prebuilt frame is rejected
   rather than silently overwritten.

Run:

```powershell
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s31_data.py -q
```

### Rollback

Restore SearchCatalog construction from `market_frames` and its unconditional
deep copy. The snapshot schema is unchanged.

### Independence

If Step 6 is absent, the eager SearchCatalog uses the one combined frame. If
Step 6 is present, the lazy SearchCatalog uses `snapshot.market_frame` on first
interaction. Neither direction requires the other.

---

## Step 6 — Build SearchCatalog on first Quick Risk or Quick Market use

### Purpose

Remove Quick Risk/Quick Market indexing from revision-1 publication. Fix 4
makes the Risk input smaller, but SearchCatalog is still not needed before a
user opens either quick-search tab.

### Files

- `rebirth/services/s02_state.py`
- `rebirth/services/s06_refresh.py`
- `rebirth/pages/risk/s08_quickrisk.py`
- `rebirth/pages/risk/s09_quickmarket.py`
- `rebirth/pages/risk/s14_workspacecallbacks.py`
- `tests/s07_integration.py`
- `tests/s10_reads.py`
- `tests/s19_riskfilters.py`

### 1. Commit without a SearchCatalog

Change the internal committed member to remain optional:

```python
self._search_catalog: SearchCatalog | None = None
self._search_catalog_revision = 0
```

During `_commit_full_snapshot()`, commit all authoritative frames first and set:

```python
self._search_catalog = None
self._search_catalog_revision = 0
```

Remove the `search_catalog` parameter from `_commit_full_snapshot()` and from
both call sites.

Remove eager `_build_snapshot_search_catalog()` calls from both:

1. the normal final financial refresh; and
2. `refresh_portfolios()`, which rebuilds mapping-dependent dashboard views.

In the normal refresh, remove the complete eager-search block: `search_started`,
the `_progress_activity(..., "search", ...)` call, the catalog build, and
`stage_durations["search"]`. Do not leave a progress stage that claims a search
index is being built. The first-use builder below owns its own timing log.

Each committed revision invalidates the previous catalog by setting the two
members above. Do not remove Risk, Market, P&L, dashboard validation, or atomic
commit.

### 2. Add one revision-keyed lazy builder

The constructor lives in `rebirth/services/s06_refresh.py`. Change its existing
threading import and initialise the new members next to the manager state:

```python
from threading import Lock, RLock

self._search_catalog_lock = Lock()
self._search_catalog_revision = 0
```

In `rebirth/services/s02_state.py`, import `monotonic` from `time` and add the
lazy method beside the existing SearchCatalog read methods:

```python
def _current_search_catalog(self) -> SearchCatalog:
    while True:
        with self._state_lock:
            snapshot = self._snapshot
            if snapshot is None or snapshot.revision <= 0:
                raise RuntimeError("No committed snapshot is available")
            revision = int(snapshot.revision)
            if (
                self._search_catalog is not None
                and self._search_catalog_revision == revision
            ):
                return self._search_catalog

        with self._search_catalog_lock:
            # Another first-use request may have built it while this request
            # waited for the dedicated search lock.
            with self._state_lock:
                snapshot = self._snapshot
                if snapshot is None or snapshot.revision <= 0:
                    raise RuntimeError("No committed snapshot is available")
                revision = int(snapshot.revision)
                if (
                    self._search_catalog is not None
                    and self._search_catalog_revision == revision
                ):
                    return self._search_catalog

                risk_frames = dict(self._risk_frames)
                market_frames = dict(self._market_frames)
                dashboard = snapshot.dashboard_frame
                market_frame = snapshot.market_frame
                risk_dates = dict(snapshot.risk_dates)
                market_date = snapshot.market_date
                market_status = snapshot.market_status

            started = monotonic()
            try:
                catalog = self._build_snapshot_search_catalog(
                    revision=revision,
                    risk_frames=risk_frames,
                    market_frames=market_frames,
                    market_frame=market_frame,
                    dashboard=dashboard,
                    risk_dates=risk_dates,
                    market_date=market_date,
                    market_status=market_status,
                )
            except Exception:
                LOGGER.exception(
                    "SearchCatalog build failed; revision=%s duration=%.3fs",
                    revision,
                    monotonic() - started,
                )
                raise
            LOGGER.info(
                "SearchCatalog build completed; revision=%s duration=%.3fs",
                revision,
                monotonic() - started,
            )

            with self._state_lock:
                current = self._snapshot
                if current is None or current.revision != revision:
                    # The context manager releases the search lock before the
                    # loop retries against the newer committed revision.
                    continue
                self._search_catalog = catalog
                self._search_catalog_revision = revision
                return catalog
```

Important: do not hold the main refresh lock while constructing the catalog.
Only copy stable references under it, build under the dedicated search lock,
then verify the revision before caching.

If Step 5 is not implemented, omit both the local
`market_frame = snapshot.market_frame` capture and the `market_frame=` builder
argument above, then use the existing `market_frames` builder. Those are the
only Step 5 compatibility lines; the rest of this chapter is unchanged.

### 3. Route all eight search reads through the lazy builder

In `rebirth/services/s02_state.py`, replace the direct
`self._search_catalog` read and the following `None` check in each of these
methods:

1. `combine_udl_options()`
2. `market_udl_options()`
3. `resolve_history_identity()`
4. `search_market_udl_options()`
5. `search_combine_udl_options()`
6. `pivot_market_exact()`
7. `pivot_combined()`
8. `pivot_combined_hierarchy()`

Each method should start its catalog work with:

```python
catalog = self._current_search_catalog()
```

Then leave the existing call on `catalog` unchanged. Do not lazy-build in only
the two dropdown methods: the pivot and history paths must work when they are
the first search operation after a refresh.

The first user interaction may take slightly longer. Every later interaction
in the same revision uses the cached catalog.

### 4. Add a tab-local build status

The existing `quick-search-data-status` and `quick-market-data-status` spans
belong to the **Open in Data** hand-off. Do not reuse them for catalog errors.

Add one new status span beside each identity dropdown:

```python
# rebirth/pages/risk/s08_quickrisk.py
html.Span(
    "",
    id="quick-risk-catalog-status",
    className="quick-search-selector-help",
    role="status",
)

# rebirth/pages/risk/s09_quickmarket.py
html.Span(
    "",
    id="quick-market-catalog-status",
    className="quick-search-selector-help",
    role="status",
)
```

In `rebirth/pages/risk/s14_workspacecallbacks.py`, add the matching status as a
third `Output` on `load_combine_udl_options()` and
`load_market_udl_options()`.

On success, return `""` for the third output. On failure, preserve the current
dropdown state and expose a short message in that tab:

```python
except (AttributeError, LookupError, TypeError, ValueError, RuntimeError) as error:
    app.logger.warning("Quick Risk catalog unavailable: %s", type(error).__name__)
    detail = " ".join(str(error).splitlines()).strip()
    message = f"Search unavailable: {type(error).__name__}: {detail}"
    return no_update, no_update, message[:300]
```

Use `"Quick Market catalog unavailable: %s"` in the market callback. The
service layer already logged the full traceback and duration, so the callback
needs only this short contextual warning. Update every other return from those
callbacks to contain three values. This makes a build failure visible without
changing the committed snapshot or either Open in Data status.

### 5. Keep search failure separate from snapshot health

If SearchCatalog construction fails:

- keep revision 1 committed and usable;
- show the error only inside Quick Risk/Quick Market;
- leave `_search_catalog` as `None` so a later interaction can retry;
- do not mark the financial refresh failed;
- do not clear the last-good snapshot.

### Tests

Add tests proving:

1. Cold refresh reaches revision 1 without calling
   `build_search_catalog()`.
2. `refresh_portfolios()` also commits without building it and invalidates a
   previously cached catalog.
3. A successful Quick Risk build is cached, so later reads in that revision do
   not build again.
4. Opening Quick Market afterward reuses it.
5. Two simultaneous first searches produce one catalog.
6. Each of the eight public search methods can be the first operation and
   obtains a catalog through the lazy builder.
7. A refresh occurring during a search build prevents the older catalog from
   being cached under the newer revision.
8. Search failure does not invalidate the dashboard revision, is visible in
   only the active quick-search tab, and a later interaction retries.

Run:

```powershell
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s19_riskfilters.py -q
python -m pytest tests/s31_data.py -q
```

### Rollback

Restore eager catalog construction before `_commit_full_snapshot()` and remove
the dedicated search lock/cache method. No connector or stored data changes.

### Independence

This step works with the existing combined MarketBook when Step 5 is absent.
It does not require Steps 1–4, 7, or 8.

---

## Step 7 — Use one owned dashboard copy during UI preparation

### Purpose

The targeted manager read already returns a private defensive copy of
`dashboard_frame`. `prepare_risk_data()` immediately copies it again before
normalising UI columns. Let it take ownership of that private read.

Fix 4 makes the dashboard much smaller, so this is no longer the first fix to
apply, but it remains safe and simple.

### Files

- `rebirth/ui/s02_aggregation.py`
- `rebirth/app/s07_factory.py`
- `rebirth/pages/risk/s15_refresh.py`
- `rebirth/pages/risk/s02_state.py`
- `tests/s06_ui.py`
- `tests/s10_reads.py`

### 1. Add an explicit ownership option

Change:

```python
def prepare_risk_data(data: pd.DataFrame) -> pd.DataFrame:
    frame = data.copy()
```

to:

```python
def prepare_risk_data(
    data: pd.DataFrame,
    *,
    take_ownership: bool = False,
) -> pd.DataFrame:
    frame = data if take_ownership else data.copy()
```

The default remains defensive for tests, static callers, and any code that may
reuse its input.

### 2. Carry ownership explicitly through the Risk cache

First, make the factory's narrow read say whether it owns the returned frame:

```python
def dashboard_frame_read() -> tuple[int, pd.DataFrame, bool]:
    reader = getattr(refresh_manager, "read_frame", None)
    if callable(reader):
        dashboard = reader("dashboard_frame")
        return int(dashboard.revision), dashboard.frame, True
    snapshot = refresh_manager.snapshot
    return int(snapshot.revision), snapshot.dashboard_frame, False
```

The `True` branch is safe because `read_frame()` already returns a deep copy.
Keep the legacy `snapshot` fallback defensive with `False`; do not guess that a
test adapter or alternate manager transfers ownership.

Update every factory unpacking, including the retry reads, from two values to
three:

```python
dashboard_revision, dashboard_frame, dashboard_owned = dashboard_frame_read()
```

For the warm initial read use:

```python
risk_data = prepare_risk_data(
    dashboard_frame,
    take_ownership=dashboard_owned,
)
```

Add `frame_is_owned: bool = False` to `prepared_committed_dashboard()`. When
that function performs `read_frame("dashboard_frame")` itself, set
`frame_is_owned = True`. When `current_cube_page()` supplies the frame, pass
the `dashboard_owned` value returned above. Prepare the non-empty frame with:

```python
prepared = prepare_risk_data(
    frame,
    take_ownership=frame_is_owned,
)
```

Keep the existing empty-frame branch; it must not call `prepare_risk_data()`,
which requires at least one row.

Next, in `rebirth/pages/risk/s02_state.py`, change
`_RiskDataCache.replace_frame()` to expose the same ownership boundary:

```python
def replace_frame(
    self,
    frame: pd.DataFrame,
    revision: int,
    *,
    take_ownership: bool = False,
) -> pd.DataFrame:
    prepared = prepare_risk_data(
        frame,
        take_ownership=take_ownership,
    )
    # Keep the existing revision check and cache publication below.
```

In `_RiskDataCache.current()` and the cold-page
`materialize_initial_dashboard()` path, the frame comes directly from
`read_frame()`, so pass ownership through the cache:

```python
return self.replace_frame(
    dashboard.frame,
    dashboard.revision,
    take_ownership=True,
)
```

```python
prepared = cache.replace_frame(
    dashboard.frame,
    dashboard.revision,
    take_ownership=True,
)
```

If a caller receives a shared reference rather than `read_frame()`, leave both
defaults at `take_ownership=False`.

This step removes one unconditional full input copy. Later pandas column
selection or assignment may still allocate smaller arrays or frames; do not
describe it as a completely allocation-free preparation.

### Tests

Add tests proving:

1. The default call does not mutate its input.
2. `take_ownership=True` returns correct values and may reuse the provided
   frame.
3. The cached prepared dashboard remains revision-keyed.
4. Risk mounting still reads only control metadata and `dashboard_frame`, as
   required by Fix 4.

Run:

```powershell
python -m pytest tests/s06_ui.py -q
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s16_refreshshell.py -q
```

### Rollback

Remove `take_ownership` and restore the unconditional copy.

### Independence

This step does not modify startup polling, P&L calculation, connector
validation, MarketBook, SearchCatalog, or table construction.

---

## Step 8 — Remove eager initial table builds

### Purpose

Avoid building the initial Risk Explorer and Aggregate P&L tables synchronously
inside `build_layout()` before their normal callbacks run.

This happens after revision 1, so it mainly addresses the apparent
"synchronising browser" stall rather than connector-stage failures.

### Files

- `rebirth/pages/risk/s16_view.py`
- `rebirth/pages/risk/s07_explorer.py`
- `rebirth/pages/risk/s14_workspacecallbacks.py`
- `tests/s06_ui.py`
- `tests/s19_riskfilters.py`

### 1. Leave callback ownership unchanged

Keep the existing Risk Explorer and Aggregate P&L callbacks as the only table
builders. They already own filtering, expansion state, selected dimension, and
the current committed filter form.

### 2. Replace eager layout tables with stable placeholders

Remove these eager calls from `build_layout()`:

```python
initial_risk_table = build_risk_table(...)
initial_aggregate_table = build_aggregate_pl_table(...)
```

Keep the same container IDs, but initialise their children with a small
placeholder:

```python
html.Div(
    "Preparing current view…",
    className="table-placeholder",
)
```

Do not change callback IDs or output properties.

Remove the now-unused `build_aggregate_pl_table` and `build_risk_table` imports
from `s16_view.py`. Keep the initial filter, `default_filtered`, initial Risk
frame, and `initial_open_rows` calculations because they still seed controls
and `open-rows-store`.

### 3. Ensure the existing callbacks run on first mount

The two normal rendering callbacks must not use `prevent_initial_call=True`.
They replace the placeholders after Dash's initial callback graph settles. More
than one initial callback execution is valid because filter stores and other
Inputs may also initialise. This step removes the two eager synchronous builds;
it does not promise exactly one callback execution.

Do not add another interval to trigger these callbacks.

### Tests

Add tests proving:

1. `build_layout()` does not call either table builder.
2. Both callback definitions remain initial-call enabled.
3. After the initial callback graph settles, both containers are no longer
   placeholders.
4. Initial callbacks render Base view automatically.
5. Activity 1–3 remain the initial Risk selection.
6. Aggregate P&L totals match the Fix 4 dashboard.
7. Chevron interactions do not cause Aggregate P&L to rebuild unnecessarily.

Run:

```powershell
python -m pytest tests/s06_ui.py -q
python -m pytest tests/s19_riskfilters.py -q
python -m pytest tests/s34_riskpivot.py -q
```

### Rollback

Restore the two eager builder calls. No financial or stored state changes.

### Independence

This step does not require the single-flight hydration lock or the UI ownership
option. It works with the current prepared dashboard and callbacks.

---

## 9. Defensive code that should not be targeted for crash reduction

The following code may look elaborate, but deleting it will not meaningfully
reduce server memory:

- the 2,400-second non-destructive startup watchdog;
- boot UUID and attempt IDs;
- progress text and stage names;
- `/healthz` and the small `/progressz` payload;
- security and no-cache response headers;
- the single-writer `RLock`;
- revision/reset-generation checks.

The watchdog only reports that a call is taking a long time; it does not retain
a DataFrame or kill the worker. Boot IDs are small strings. Health and progress
endpoints return scalar metadata.

They may be simplified later for maintainability, but that should be a separate
cosmetic commit with no claim that it fixes memory crashes.

On a true process cold start there is no previous revision-1 financial
snapshot to retain. Last-good coexistence matters during a warm/manual refresh,
when the committed snapshot remains readable while the next one is staged.
That is intentional service continuity. Keep it, and reduce the temporary copy
chain with the independent steps above instead.

Do not delete:

- connector schema validation;
- numeric and nonblank validation;
- Open/Current uniqueness checks;
- tenor-order authority checks;
- `one_to_one` Open/Current merge validation;
- `many_to_one` Risk/Market merge validation;
- the strengthened Fix 4 dashboard aggregation validation;
- Portfolio and Reported Underlying mapping;
- RiskChecker readiness `Age`;
- atomic commit and last-good retention.

Also do not delete the committed `_pl_frames`, `_risk_frames`, or
`_market_frames` merely because equivalent-looking published frames exist.
`refresh_portfolios()` reuses detailed P&L, partial refreshes reuse unchanged
product frames, and Step 6's lazy SearchCatalog needs the committed Risk/Market
sources. Removing those caches is a separate refresh-architecture change, not
a defensive-code cleanup.

Fix 4 makes `_validate_dashboard_release()` operate on the smaller,
Portfolio-free frame and strengthens it to prove the collapse was correct.
Deleting it wholesale now saves little and can hide a financially incorrect
aggregation.

## 10. Optional feeds are not defensive-code deletions

Cross Gamma and New Trades are loaded during revision 1 and can expand the
Market scope. They can be moved to a second revision only if the business
accepts a first snapshot that temporarily excludes those rows.

Similarly:

- disabling RiskChecker changes Risk-date selection unless `Age=0` is valid;
- excluding Commodity Risk removes Commodity positions;
- deferring baseline promotion can change Display Bucket and Top Promotions;
- removing Reported Underlying changes reporting identity.

Those are product-scope decisions, not safe defensive cleanup. Keep them out of
the independent Fix 10 patches unless their semantics are agreed separately.

## 11. Apply any subset safely

For any chosen subset—for example Steps 2, 5, and 6—use one commit per step:

```text
fix10/step2-pl-input-ownership
fix10/step5-marketbook-reuse
fix10/step6-lazy-search
```

After each step:

1. Start from a clean worktree.
2. Apply only that chapter.
3. Run its focused tests.
4. Run the full suite.
5. Run one real cold start.
6. Compare totals and grain with the pre-step snapshot.
7. Record timing and peak RSS.
8. Keep or revert that one commit before evaluating the next step.

Final gates for any subset:

```powershell
python -m pytest -q
python -m ruff check .
python -m ruff format --check .
git diff --check
```

Financial comparison:

```python
before_dashboard = before.dashboard_frame
after_dashboard = after.dashboard_frame

assert "Portfolio" not in after_dashboard
assert "Portfolio" in after.combined_pl

for column in ["Risk", "dRisk", "PL"]:
    assert after_dashboard[column].sum(min_count=1) == pytest.approx(
        before_dashboard[column].sum(min_count=1)
    )
```

Operational verification:

1. Cold start reaches revision 1.
2. `/healthz` remains responsive during startup.
3. `/progressz` remains frame-free and responsive.
4. The Python process does not restart.
5. Risk automatically opens on the Base view.
6. Stock and P&L retain Portfolio filters.
7. Quick Risk and Quick Market work when opened.
8. A failed refresh retains the last good revision.

If failures still move randomly between Market Underlyings, treat that as a
connector-call-volume, rate-limit, or I/O problem. Use the appropriate bulk
Market guide and connector-owned timeout/retry policy. Fix 10 primarily reduces
Python memory peaks, CPU-heavy post-processing, and duplicate browser work.
