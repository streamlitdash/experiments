# The Fix — Rebirth V4.1 cold-start simplification

This is the consolidated implementation guide for the Rebirth V4.1 cold-start problem.

It combines the strongest parts of `V1.md`, `V2.md`, `FIX10.md`, `FIX4.md`, `FIX1.md`, `FIX6.md`, `STEP3.md`, and the previous `cold-start-change` branch, but corrects the parts that do not safely apply to the current `main` branch.

The intended baseline is Rebirth V4.1 `main` around commit `cdf1e770f57eeb25e5cd5bd5a8620997ed1f90ee`.

The objective is to reduce:

- cold-start wall-clock time;
- peak RSS and temporary DataFrame duplication;
- duplicate browser hydration work;
- unnecessary connector pressure;
- work performed before revision 1 becomes usable;

without weakening:

- connector validation;
- position and quote identity;
- P&L formulas;
- Portfolio and Reported Underlying authority;
- RiskChecker date selection;
- atomic publication;
- last-good revision retention;
- stale revision/reset-generation protection;
- missing-market semantics.

This guide is deliberately modular. Implement and measure each phase independently. Do not combine several phases into one unmeasured commit.

---

# 0. What is actually wrong

The process-wide startup coordinator, refresh lock, revision checks, watchdog, and last-good snapshot logic are not the main source of normal-path cold-start cost.

The expensive behaviour is around them:

1. broad snapshot reads copy several large financial frames when the caller needs only control metadata or `dashboard_frame`;
2. the same dashboard can be prepared more than once for the same revision;
3. UI preparation repeats validation and large-frame normalization already proven before commit;
4. `get_product_pl()` copies validated Risk and Market frames immediately before `pandas.merge()` allocates a new result;
5. connector validation can pass through chains of full-frame copies;
6. the combined MarketBook is built for the snapshot and effectively rebuilt/copied again for Search;
7. SearchCatalog is built before revision 1 is published even when the user never opens Quick Risk or Quick Market;
8. `build_layout()` synchronously constructs the initial Risk table and Aggregate P&L table before the full page can return;
9. a revision may become visible while the startup worker still retains its final large local frames;
10. manager-wide Market retry/concurrency policy can amplify connector pressure during failures;
11. the current Risk analytical dashboard still remains at Portfolio grain, so every downstream UI operation carries far more rows than necessary if Portfolio is not required in Risk analytics.

The existing commit that filters zero-click Dash remount events is valid and should remain. It prevents a remounted control from starting another manual refresh, but it does not solve duplicate cold-start hydration or memory amplification.

---

# 1. Non-negotiable protections

Do not remove or weaken any of the following while implementing this guide:

- one process-wide refresh writer;
- `StartupCoordinator` ownership of revision 1;
- atomic snapshot publication;
- last-good snapshot retention;
- `expected_revision` checks;
- reset-generation checks;
- one-to-one Market quote identity validation;
- many-to-one Risk-to-Market merge validation;
- Risk position uniqueness;
- connector-owned tenor ordering;
- Portfolio mapping authority;
- Reported Underlying mapping authority;
- RiskChecker `Age` and forced-date behaviour;
- market status routing (`Live` versus `OFFICIAL`);
- missing-market rows remaining unavailable rather than becoming zero;
- quote aggregation rules that prevent Portfolio count from weighting Market values;
- connector numeric, schema, and nonblank-key validation;
- archive integrity rules.

The goal is to remove duplicate ownership work, not financial correctness checks.

---

# 2. Measurement gate

Before every phase, run the same production-shaped cold start and record:

```text
cold-start total seconds
peak process RSS
revision reached
Risk rows loaded
combined_pl rows
dashboard_frame rows
Market rows
Market Open connector attempts
Market Current connector attempts
Risk connector attempts
release seconds
Search seconds
dashboard preparation seconds
initial layout seconds
time from revision 1 publication to usable Risk page
```

Also record whether a failure is:

```text
process exited / OOM killed
process alive but HTTP unresponsive
connector call stalled
revision 1 published but browser still synchronising
```

Do not judge a patch only from the browser reconnect message.

---

# 3. Phase 1 — Replace broad cold-start reads with narrow reads

This is the first change to make on current `main`.

The old `cold-start-change` branch was directionally correct here, but it is stale. Reimplement the pattern on current `main`; do not merge that branch wholesale.

## Purpose

The Risk page needs:

- compact committed control state; and
- one committed `dashboard_frame`.

It does not need a broad `RefreshSnapshot` copy containing:

- combined P&L;
- MarketBook;
- dashboard;
- unmapped rows;
- other large frames.

The persistent refresh shell needs only compact control metadata.

## Files

- `rebirth/app/s07_factory.py`
- `rebirth/pages/risk/s15_refresh.py`
- `rebirth/app/s02_contracts.py` if protocol typing needs adjustment
- relevant startup and read tests

## 3.1 Use `control_snapshot` for UI control metadata

Every manager-backed path that needs only:

- revision;
- refreshed time;
- reason;
- dates;
- settings;
- readiness;

must use:

```python
refresh_manager.control_snapshot
```

Do not call `refresh_manager.snapshot` for the shared refresh shell.

Do not add a production fallback that silently reads the broad snapshot when `control_snapshot` is missing. The real manager protocol already owns this API. Test adapters should be updated explicitly.

## 3.2 Add one narrow dashboard reader in the factory

Use a helper shaped like:

```python
def dashboard_frame_read() -> tuple[int, pd.DataFrame]:
    dashboard = refresh_manager.read_frame("dashboard_frame")
    return int(dashboard.revision), dashboard.frame
```

`read_frame()` returns one defensive frame copy rather than a broad snapshot copy.

## 3.3 Read one revision-consistent pair

For warm startup and page mounting:

```python
snapshot = refresh_manager.control_snapshot
dashboard_revision, dashboard_frame = dashboard_frame_read()

if dashboard_revision != int(snapshot.revision):
    snapshot = refresh_manager.control_snapshot
    dashboard_revision, dashboard_frame = dashboard_frame_read()

if dashboard_revision != int(snapshot.revision):
    raise RuntimeError(
        "Could not read one consistent committed dashboard revision"
    )
```

Retry once because a refresh may commit between the two narrow reads. Do not loop indefinitely.

## 3.4 Remove the broad Risk snapshot cache

Do not retain a `risk_snapshot_cache` containing a full copied snapshot.

Keep compact control metadata and the prepared dashboard cache separately.

## 3.5 Cold Risk materialisation must use the narrow dashboard read

Change cold hydration so it receives compact control state and explicitly reads the dashboard:

```python
def materialize_initial_dashboard(control_snapshot):
    dashboard = refresh_manager.read_frame("dashboard_frame")
    if int(dashboard.revision) != int(control_snapshot.revision):
        control_snapshot = refresh_manager.control_snapshot
        dashboard = refresh_manager.read_frame("dashboard_frame")
    if int(dashboard.revision) != int(control_snapshot.revision):
        raise RuntimeError(
            "Could not read one consistent initial dashboard revision"
        )

    prepared = cache.replace_frame(
        dashboard.frame,
        dashboard.revision,
        take_ownership=True,
    )
    return build_layout(
        prepared,
        control_snapshot,
        refresh_enabled=True,
        stage_delays=refresh_manager.stage_delays,
        include_shared_refresh_shell=False,
    )
```

The `take_ownership=True` argument is introduced in Phase 4 below. If Phase 4 has not yet landed, omit that argument temporarily and keep the normal defensive copy.

## Tests

Prove:

1. Cold Risk hydration does not call the broad `snapshot` property.
2. Shared-shell hydration does not call the broad `snapshot` property.
3. Warm Risk mounting reads only `control_snapshot` and `dashboard_frame`.
4. A revision change between the two narrow reads retries once and returns a consistent revision.
5. A second mismatch fails closed.
6. Existing static-data mode remains unchanged.

## Rollback

Restore broad reads only if the narrow-read tests expose an actual protocol violation. Do not keep both paths in production.

---

# 4. Phase 2 — Do not declare startup complete while the writer still owns large frames

Current startup can expose revision 1 before the startup worker has actually returned from `refresh()`. That allows browser hydration to begin while the refresh stack can still retain large local DataFrames.

This phase must be implemented after Phase 1 so cold hydration no longer depends on a broad snapshot.

## Files

- `rebirth/app/s04_startup.py`
- `rebirth/pages/risk/s15_refresh.py`
- `rebirth/ui/s04_components.py`
- `tests/s12_startup.py`
- `tests/s16_refreshshell.py`

## 4.1 Add a real writer-completion signal

Do not infer exact worker completion only from `manager.progress.running`.

The refresh pipeline currently calls `_finish_progress()` before the Python function has fully returned, so `progress.running == False` is not a perfect proxy for stack/local-frame release.

Add an explicit process-owned lifecycle signal. One acceptable design is a manager property such as:

```python
@property
def writer_active(self) -> bool:
    ...
```

Set it before entering the refresh transaction and clear it in the outermost `finally` after the refresh function has completed all post-commit work.

Alternatively, expose an equivalent exact refresh-returned generation/state from the coordinator/manager. The important contract is:

```text
revision > 0 does not imply startup worker has returned
```

## 4.2 Correct `StartupCoordinator.start()`

The positive-revision shortcut must not force `phase="succeeded"` while an owned writer is still active.

Use the exact lifecycle signal plus worker liveness.

Conceptually:

```python
worker_alive = self._worker is not None and self._worker.is_alive()
writer_active = self._writer_active()

if self._revision() > 0:
    if not worker_alive and not writer_active:
        self._phase = "succeeded"
        self._error = None
    return False
```

## 4.3 Apply the same rule to `schedule_start()` and `status()`

Do not fix only one coordinator entry point.

All positive-revision shortcuts must use the same completion rule.

`_run()` remains the normal owner of `_succeed()` after `refresh()` returns.

Keep `copy_result=False` for startup.

## 4.4 Correct external-writer following

When this coordinator encounters an already-running refresh, it must wait for:

```text
revision > 0
and exact writer lifecycle stopped
```

before calling `_succeed()`.

If the external writer stops without a revision, fail the attempt.

Do not start a second writer merely because the watchdog fired.

## 4.5 Add one process-level hydration single-flight lock

In `register_refresh_callbacks()`:

```python
from threading import Lock

startup_hydration_lock = Lock()
```

When startup is truly complete:

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

This limits heavy Python materialisation to one request at a time per process.

It is a concurrency cap, not a lifetime once-only guarantee. A later request can still use the prepared revision cache.

## 4.6 Slow only the cold-page poll

Change:

```python
interval=500
```

to:

```python
interval=1000
```

for the lightweight cold-page startup interval.

The completion gate and hydration lock provide the actual protection; this change simply halves polling traffic.

## Tests

Prove:

1. Revision 1 can be visible while startup still reports running.
2. `start()`, `schedule_start()`, and `status()` all obey the same rule.
3. An external writer is followed until exact lifecycle completion.
4. An owned worker reports success after `refresh()` returns.
5. Two simultaneous hydration callbacks never execute heavy materialisation concurrently.
6. A skipped caller can succeed on a later poll.
7. A failed initial refresh still exposes Retry.
8. Last-good committed data remains available after later refresh failures.

---

# 5. Phase 3 — Remove the two unnecessary P&L input copies

This is the smallest high-value calculation change.

## Files

- `rebirth/domain/s03_calculations.py`
- `tests/s04_market.py`
- integration tests

## Change

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

Keep:

```python
result = risk.merge(
    market,
    on=spec.market_keys,
    how="left",
    validate="many_to_one",
    indicator="_market_merge",
)
```

Do not mutate `risk` or `market` afterward. All assignments remain on `result` or later derived copies.

## Tests

```python
expected_risk = risk.copy(deep=True)
expected_market = market.copy(deep=True)

result = get_product_pl(...)

pd.testing.assert_frame_equal(risk, expected_risk)
pd.testing.assert_frame_equal(market, expected_market)
assert result is not risk
assert result is not market
```

Run the Market and integration suites.

---

# 6. Phase 4 — Use one revision-aware prepared dashboard

The current system can prepare the same dashboard more than once and may perform the expensive preparation before discovering that a revision is already cached.

This phase combines the strongest part of V2 with the ownership idea from FIX10 Step 7.

## Files

- `rebirth/ui/s02_aggregation.py`
- `rebirth/pages/risk/s02_state.py`
- `rebirth/app/s07_factory.py`
- `rebirth/pages/risk/s15_refresh.py`
- tests for UI reads/cache behaviour

## 6.1 Add explicit ownership to `prepare_risk_data()`

Change:

```python
def prepare_risk_data(data: pd.DataFrame) -> pd.DataFrame:
    frame = data.copy()
```

into:

```python
def prepare_risk_data(
    data: pd.DataFrame,
    *,
    take_ownership: bool = False,
) -> pd.DataFrame:
    frame = data if take_ownership else data.copy()
```

Default remains defensive for:

- static callers;
- tests;
- arbitrary external DataFrames.

Use `take_ownership=True` only for a frame returned directly from `read_frame("dashboard_frame")`, because that read is already a private defensive copy.

## 6.2 Fix `_RiskDataCache.replace_frame()` ordering

Do not prepare first and check the revision afterward.

Use a double-check pattern:

```python
def replace_frame(
    self,
    frame: pd.DataFrame,
    revision: int,
    *,
    take_ownership: bool = False,
) -> pd.DataFrame:
    selected_revision = int(revision)

    with self._lock:
        if selected_revision <= self._revision:
            return self._frame

    prepared = prepare_risk_data(
        frame,
        take_ownership=take_ownership,
    )

    with self._lock:
        if selected_revision <= self._revision:
            return self._frame

        self._frame = prepared
        self._revision = selected_revision
        self._filtered.clear()
        self._rendered.clear()
        self._promotion_generations.clear()
        return prepared
```

The first check prevents an ordinary duplicate request from preparing millions of rows needlessly.

The second check handles the race where another request publishes the revision while this request is preparing.

## 6.3 Narrow reads transfer ownership explicitly

When `_RiskDataCache.current()` performs:

```python
dashboard = manager.read_frame("dashboard_frame")
```

use:

```python
return self.replace_frame(
    dashboard.frame,
    dashboard.revision,
    take_ownership=True,
)
```

Do the same for cold hydration.

## 6.4 Consolidate duplicate prepared-dashboard caches

Do not keep separate long-lived prepared copies in both:

- `s07_factory.py`; and
- `_RiskDataCache`.

Choose one process-owned revision-aware prepared cache as the authority for Risk and P&L page preparation.

The factory may hold a lightweight reference to the authoritative cache, but it should not independently retain another prepared DataFrame for the same revision.

A reasonable architecture is:

```text
manager committed dashboard_frame
        |
        v
one process-owned PreparedDashboardCache
        |
        +--> Risk page
        +--> Aggregate P&L page
        +--> callbacks
```

The cache key is the committed financial revision.

## Tests

Prove:

1. Default `prepare_risk_data()` does not mutate caller input.
2. `take_ownership=True` produces the same values.
3. A duplicate revision short-circuits before calling `prepare_risk_data()`.
4. A stale request that loses the race does not replace a newer prepared revision.
5. Risk and P&L reuse one prepared revision.
6. Cache invalidation occurs on a new committed revision.

---

# 7. Phase 5 — Reuse one combined MarketBook

Current refresh publishes a combined MarketBook and then Search reconstructs/copies an equivalent Market representation from per-product Market frames.

Stop doing both.

## Files

- `rebirth/services/s06_refresh.py`
- `rebirth/domain/s10_search.py`
- `rebirth/services/s02_state.py`
- integration and search tests

## 7.1 Build `combined_market` once

In the refresh finalisation path:

```python
combined_market = self._combined_market_frame(
    next_market,
    market_date,
)
```

Assign this exact frame to:

```python
snapshot.market_frame
```

Do not call `_combined_market_frame()` again for the same financial refresh.

Portfolio-only refreshes should reuse the committed MarketBook and call `_combined_market_frame()` zero times.

## 7.2 Let Search accept an already combined MarketBook

Extend `build_search_catalog()` with optional arguments:

```python
def build_search_catalog(
    *,
    revision: int,
    risk_frames: Mapping[str, pd.DataFrame],
    risk_pivot_frame: pd.DataFrame | None = None,
    market_frame: pd.DataFrame | None = None,
    market_frames: Mapping[str, pd.DataFrame] | None = None,
    risk_dates: Mapping[str, pd.Timestamp],
    market_date: pd.Timestamp,
) -> SearchCatalog:
    ...
```

Keep `market_frames` for direct-library/test compatibility.

If `market_frame` is present, project it rather than concatenating every product again.

## 7.3 Add a narrow projection helper

Use:

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

Do not silently overwrite a stale/missing Market Date.

## 7.4 Avoid a second SearchCatalog Market deep copy on the owned builder path

Give `SearchCatalog.__init__()` a defensive default:

```python
copy_market_frame: bool = True
```

and use:

```python
self._market_frame = (
    market_frame.copy(deep=True)
    if copy_market_frame
    else market_frame
)
```

The normal `build_search_catalog()` path owns its narrow Market projection, so it may pass `copy_market_frame=False`.

Direct public construction keeps the defensive copy.

## Tests

Prove:

1. `_combined_market_frame()` executes once per financial refresh.
2. Portfolio-only refresh executes it zero times.
3. Quick Market results are unchanged.
4. Search cannot mutate `snapshot.market_frame`.
5. Direct SearchCatalog construction remains defensive.
6. Mixed, missing, or stale Market Date is rejected.

---

# 8. Phase 6 — Build SearchCatalog on first use

Search is not required before the Risk page is usable.

Move SearchCatalog construction out of revision-1 publication.

## Files

- `rebirth/services/s02_state.py`
- `rebirth/services/s06_refresh.py`
- `rebirth/pages/risk/s08_quickrisk.py`
- `rebirth/pages/risk/s09_quickmarket.py`
- `rebirth/pages/risk/s14_workspacecallbacks.py`
- search/integration tests

## 8.1 Commit revision 1 without SearchCatalog

Internal state:

```python
self._search_catalog: SearchCatalog | None = None
self._search_catalog_revision = 0
```

Every committed financial or Portfolio-only revision invalidates Search:

```python
self._search_catalog = None
self._search_catalog_revision = 0
```

Remove SearchCatalog from `_commit_full_snapshot()` arguments.

Remove eager search construction from:

- normal financial refresh;
- Portfolio-only refresh.

Do not leave a refresh progress stage claiming Search is being built.

## 8.2 Add a dedicated revision-keyed search lock

In manager construction:

```python
from threading import Lock, RLock

self._search_catalog_lock = Lock()
```

## 8.3 Add one lazy builder

Use this structure:

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
                    continue
                self._search_catalog = catalog
                self._search_catalog_revision = revision
                return catalog
```

Do not hold the main refresh lock during Search construction.

## 8.4 Route every Search read through the lazy builder

All of these must work when they are the first Search operation of a revision:

1. `combine_udl_options()`
2. `market_udl_options()`
3. `resolve_history_identity()`
4. `search_market_udl_options()`
5. `search_combine_udl_options()`
6. `pivot_market_exact()`
7. `pivot_combined()`
8. `pivot_combined_hierarchy()`

Each starts with:

```python
catalog = self._current_search_catalog()
```

## 8.5 Keep Search failure separate from financial snapshot health

If Search build fails:

- revision 1 remains committed and usable;
- Risk Explorer remains usable;
- P&L remains usable;
- `_search_catalog` remains `None`;
- the next Search interaction may retry;
- the error is shown only in Quick Risk/Quick Market context;
- the refresh is not marked failed.

Add dedicated small status spans for Quick Risk and Quick Market catalog construction. Do not reuse the Open-in-Data status controls.

## Tests

Prove:

1. Cold refresh reaches revision 1 without Search construction.
2. Portfolio refresh also commits without Search construction.
3. First Search builds once.
4. Later Search calls in the same revision reuse it.
5. Two simultaneous first Search requests build one catalog.
6. Every public Search method can be first.
7. A refresh during Search build prevents stale catalog publication.
8. Search failure does not invalidate financial state.

---

# 9. Phase 7 — Defer initial Risk and Aggregate P&L table builds

Revision 1 should not have to synchronously build large HTML component trees before the page can return.

## Files

- `rebirth/pages/risk/s16_view.py`
- Risk Explorer callback files
- Aggregate P&L callback files
- UI tests

## 9.1 Remove the two eager builders from `build_layout()`

Remove synchronous calls to:

```python
build_risk_table(...)
build_aggregate_pl_table(...)
```

Keep the control setup and `initial_open_rows` calculation if they are still required to seed stores.

## 9.2 Keep stable output containers with placeholders

For `risk-grid` and `aggregate-pl-grid`, initialise children with a small placeholder such as:

```python
html.Div(
    "Preparing current view…",
    className="table-placeholder",
)
```

Keep the existing container IDs unchanged.

## 9.3 Existing callbacks become the only table builders

The normal Risk Explorer and Aggregate P&L callbacks must remain initial-call enabled.

Do not add a new interval just to trigger table construction.

More than one initial callback execution may still occur because stores initialise independently. The revision-aware data/render caches should make duplicate executions cheap.

## Tests

Prove:

1. `build_layout()` no longer calls either table builder.
2. Both table callbacks remain initial-call enabled.
3. Placeholders are replaced automatically after the callback graph settles.
4. Initial Risk type and default filters remain unchanged.
5. Aggregate P&L totals are identical.
6. Chevron interactions do not rebuild unrelated Aggregate P&L unnecessarily.

---

# 10. Phase 8 — Reduce connector validation copy chains

Do this after the earlier hydration/Search changes have been measured.

The purpose is not to remove validation. It is to validate one owned working frame instead of copying the same connector result repeatedly.

## Files

- `rebirth/domain/s03_calculations.py`
- governance callers that import shared helpers
- adapter and integration tests

## 10.1 Keep one ownership copy at `_load_frame()`

Keep:

```python
return frame.copy()
```

That boundary protects connector-owned or cached data.

After this point, the pipeline owns the returned DataFrame.

## 10.2 Add `owned` flags to shared mutation helpers

Example:

```python
def _require_nonblank(
    frame: pd.DataFrame,
    columns: list[str],
    label: str,
    *,
    owned: bool = False,
) -> pd.DataFrame:
    result = frame if owned else frame.copy()
    ...
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
    ...
```

Default remains defensive for governance/public callers.

Pass `owned=True` only from validation chains operating on the frame returned by `_load_frame()`.

## 10.3 Private pipeline validators may operate on owned input

For private helpers such as:

- `_enforce_product()`;
- `_validate_market_tenor_orders()`;

replace unnecessary immediate copies only when their input is guaranteed to be pipeline-owned.

Do not remove any checks.

## 10.4 Avoid final projection copies only at proven ownership boundaries

When the working frame is private and no caller-shared backing frame can be mutated, use a direct column projection rather than another unconditional deep copy.

Do not apply this mechanically to public helpers.

## Tests

For Risk, Open, Current, and governance merge paths:

1. Deep-copy the caller source.
2. Run the public validator.
3. Assert caller source unchanged.
4. Assert returned canonical dtypes and columns unchanged.
5. Assert all previous invalid input cases still fail.

---

# 11. Phase 9 — Replace universal connector retries with connector-owned policy

Current manager-level Market concurrency can improve normal latency, but a universal policy of many workers plus four retries can make outage behaviour much worse.

Do not simply remove concurrency. Make the connector policy explicit.

## 11.1 Keep scalar fallback but prefer explicit bulk hooks

Extend `ProductConnectorAdapter` with optional hooks such as:

```python
market_open_bulk
market_status_bulk
```

The bulk contract receives:

```python
(date, tuple_of_underlyings, market_status=...)
```

and returns one DataFrame covering that scope.

Do not overload the existing scalar connector with a value that is sometimes text and sometimes a tuple.

## 11.2 Roll bulk out per Source Type

For a selected source:

```python
if adapter.market_open_bulk is not None:
    ...
else:
    # existing per-Underlying fallback
```

Same for Current.

The returned bulk frame must still pass through:

- `get_product_market_open()`;
- `get_product_market_status()`;
- `_reject_unrequested_market_underlyings()`.

Bulk loading does not bypass validation.

## 11.3 Put real I/O timeouts in the connector/client

A Python thread timeout around `future.result()` does not terminate a stuck underlying call.

Use the native timeout option of the real HTTP/MRX/database client.

## 11.4 Retry only explicitly transient failures

Do not use one broad manager-wide:

```python
except Exception:
    retry
```

for production connectors.

Each connector should declare or translate transient failures such as:

- temporary network reset;
- timeout;
- service unavailable;
- throttling when retry is permitted.

Permanent failures such as:

- authentication;
- permission;
- bad schema;
- programming errors;
- invalid request;

must fail fast.

## 11.5 Use bounded exponential backoff with jitter

Prefer:

```text
attempt 1
short jittered backoff
attempt 2
larger jittered backoff
...
total connector deadline reached -> fail
```

rather than synchronized fixed 0.5-second retry waves.

## 11.6 Make metrics truthful

For bulk hooks, count one logical Open call and one logical Current call for the complete selected scope.

Record retry attempts separately.

Useful metrics:

```text
source_type
operation
logical_calls
attempts
retries
timeouts
rows
duration
```

Do not log position identities or confidential values in generic telemetry.

## 11.7 Risk concurrency needs the same discipline

Product-level Risk concurrency is useful only for genuinely independent adapter calls.

One large Credit adapter remains one task unless the connector itself owns several real non-overlapping requests.

Do not call the same Risk query ten times and deduplicate afterward.

Do not stack a ten-worker manager pool on top of ten-worker adapter pools without calculating total concurrency.

---

# 12. Phase 10 — Optional: remove calculation-level global sorting

Do not make this an early cold-start fix.

Current `get_product_pl()` sorts the complete Portfolio-grain result. At production scale that can allocate large index arrays and consume meaningful CPU/RSS.

However, removing it changes the output-row-order contract.

Only proceed after measuring the sort as a significant cost and auditing every consumer for row-position assumptions.

If implemented:

- compare financial results by complete identity, not row order;
- keep Market tenor ordering as presentation authority;
- keep Gamma sourced/derived row ordering if required;
- audit `drop_duplicates(keep="first")` consumers;
- audit archive reproducibility requirements;
- audit dropdown ordering and external integrations.

This should be a standalone behavioural PR with focused tests and easy rollback.

---

# 13. Separate business/data-grain change — remove Portfolio from the Risk analytical layer

This is the major row-count reduction described by `FIX4.md`.

Do not mix it into the technical cold-start PRs above.

Current `main` still publishes `dashboard_frame` with Portfolio. Therefore the old FIX10 assumption:

```python
assert "Portfolio" not in snapshot.dashboard_frame
```

is not true on current `main`.

If the product decision is that Portfolio is not required in Risk analytics, implement Fix 4 separately.

## Correct grain boundaries

Portfolio must remain in:

```text
Raw Risk connector
Product P&L
combined_pl
Portfolio mapping
Reported Underlying mapping
Unmapped diagnostics
Stock
P&L workflows
adjustments/sending
official Portfolio-grain Risk archive
```

Portfolio may be removed only after P&L and governance have been calculated for the Risk analytical projection.

Required flow:

```text
Risk at Portfolio grain
  -> Market join
  -> P&L
  -> Portfolio mapping
  -> Reported Underlying
  -> baseline promotions
  -> combined_pl remains detailed
  -> mapped Risk analytical rows collapse across Portfolio
  -> Portfolio-free dashboard_frame
  -> Risk Explorer / Aggregate P&L / Quick Risk
```

## Financial aggregation rules

When collapsing Portfolio:

- Risk, dRisk, P&L and registered Credit measures are additive;
- all-missing nullable metrics remain missing;
- Open and Current are carried once only after proving repeated Portfolio rows agree;
- Move is recomputed as `Current - Open`;
- Market availability/status must remain consistent;
- Promotion Score and Vol Score use a ranking-safe rule such as `max`, never sum;
- thresholds and governance values must be proven identical within the collapsed grain;
- every other analytical identity field remains part of the group key.

Do not use `drop_duplicates()` as a substitute for aggregation.

Do not sum Open or Current.

## Product/UI consequences requiring explicit sign-off

Risk would lose Portfolio from:

- Risk filters;
- Risk View-by;
- Quick Risk pivot/filter dimensions.

Stock and P&L would keep Portfolio.

Saved-view compatibility must preserve hidden Portfolio selections for Stock/P&L even if Risk only displays a four-field projection.

Official Risk history must remain Portfolio grain if that is the governed archive contract.

## Tests

Before merging this business change, reconcile:

```text
combined_pl totals before/after
Risk analytical totals before/after
P&L totals before/after
per Risk Type / Risk Greek / Underlying / tenor totals
Portfolio P&L workflow
saved views
Risk archive output
Quick Risk
Quick Market
Stock
Unmapped books
```

This change can produce the largest row-count reduction, but it is not merely a performance refactor.

---

# 14. Optional later UI optimisation — trusted committed-manager preparation

Even after removing the first defensive input copy, `prepare_risk_data()` still performs work that is valuable for arbitrary DataFrames but redundant for a manager-committed dashboard:

- numeric conversion checks;
- missing/non-finite checks;
- breakdown checks;
- tenor-order reconciliation;
- repeated full-frame normalization.

After the earlier phases are measured, consider a separate trusted manager-owned preparation path.

The public/default path remains fully defensive.

The trusted path is allowed only when the source is a committed `dashboard_frame` from the refresh manager at a proven revision.

It should perform only transformations still required for display.

In particular, if tenor-order fallback remains expensive, resolve missing orders at quote-grain unique keys and map them back to positions rather than repeatedly scanning Portfolio-grain rows.

Do not weaken ingestion/release validation merely because the UI has a trusted path.

---

# 15. Optional later optimisation — promotion and hierarchy indexes

If initial Risk rendering remains expensive after:

- narrow reads;
- one prepared cache;
- lazy Search;
- deferred initial tables;
- Portfolio collapse if approved;

then profile hierarchy construction.

Possible remaining cost:

- repeatedly filtering the complete frame for promoted underlyings;
- eager construction of Market metric hierarchy indexes that the shallow first Risk view does not use.

Safe later optimisations include:

1. build reusable grouped promotion indexes;
2. build Open/Current/Move hierarchy indexes lazily on first `include_market=True` request;
3. retain the quote-level aggregation rules that prevent Portfolio weighting.

Do not change initial open/closed Risk behaviour merely to hide performance cost unless that is a separate product decision.

---

# 16. Recommended implementation order

Implement in this exact order unless measurement gives a strong reason to change it.

## PR 1 — Narrow startup reads

- `control_snapshot` for control state;
- `read_frame("dashboard_frame")` for Risk data;
- shared shell uses compact control state;
- remove broad Risk snapshot cache;
- revision-consistent narrow reads.

Expected effect: very high peak-memory reduction during hydration.

## PR 2 — Exact completion + hydration single-flight

- do not report startup success while writer still active;
- expose exact writer lifecycle;
- one non-blocking hydration lock;
- cold poll to 1 second.

Expected effect: prevents overlapping refresh locals and UI hydration peaks.

## PR 3 — Remove two P&L pre-merge copies

Expected effect: high at detailed Portfolio grain; very low semantic risk.

## PR 4 — One prepared-dashboard cache

- precheck revision before prepare;
- postcheck before publish;
- ownership transfer for narrow reads;
- one cache shared by Risk/P&L.

Expected effect: high when duplicate browsers/callbacks occur; significant peak-memory reduction.

## PR 5 — Reuse combined MarketBook

Expected effect: medium; removes duplicate Market assembly/copying.

## PR 6 — Lazy SearchCatalog

Expected effect: medium-to-high improvement in time to usable revision 1; first Quick Search interaction becomes slower instead.

## PR 7 — Defer initial Risk and Aggregate P&L tables

Expected effect: improves revision-1-to-browser usability and reduces synchronous response work.

## PR 8 — Connector-specific bulk/timeouts/retries

Expected effect: potentially very high connector latency improvement and much safer outage behaviour.

## PR 9 — Owned validator chain

Expected effect: high only if connector validation remains a large RSS/time component after the earlier fixes.

## PR 10 — Optional sort removal

Only if measured and all row-order consumers have been audited.

## Separate product PR — Portfolio-free Risk dashboard

Potentially the largest steady-state row-count reduction, but requires financial/product sign-off.

---

# 17. Acceptance criteria for the combined fix

The technical cold-start work is successful only if all of the following remain true.

## Correctness

```text
same Risk values by canonical identity
same dRisk values by canonical identity
same P&L values by canonical identity
same Market Open/Current/Move values
same missing-market behaviour
same Market Status routing
same Risk dates and forced-date behaviour
same Portfolio mapping
same Reported Underlying mapping
same promotion results unless a separate promotion patch intentionally changes them
same archive contracts
```

## Atomicity and failure behaviour

```text
no partial revision publication
last-good revision remains readable after failure
stale revision/reset generation is rejected
only one startup writer exists per process
Search failure cannot invalidate financial state
UI preparation failure cannot corrupt manager state
```

## Performance

At production shape, demonstrate measurable improvement in:

```text
peak RSS
cold-start total seconds
time to revision 1
time from revision 1 to usable Risk page
number of broad financial copies
Search work before first page
duplicate dashboard preparations
connector attempts during failure
```

## Operational behaviour

`/healthz` and `/progressz` must remain lightweight and reachable whenever the Python process is alive and not CPU-starved by a heavy synchronous callback.

A watchdog warning must never launch a second writer over a still-running connector call.

---

# 18. Things not to merge wholesale

Do not merge any of the following as one unreviewed patch:

- the old `cold-start-change` branch;
- all of FIX10 on current `main`;
- FIX4 mixed into the technical performance changes;
- manager-wide blanket retry rules as the final production connector architecture;
- global sort removal without consumer audit;
- broad snapshot compatibility fallbacks in the real manager path.

Use the old documents as source material, not as branch-sized patches.

---

# 19. Short version

If only the highest-value safe work can be done first, do these six things:

1. Replace every cold Risk/shared-shell broad snapshot read with `control_snapshot` plus targeted `read_frame("dashboard_frame")`.
2. Do not let the startup coordinator report success until the refresh writer has actually returned, then single-flight heavy hydration.
3. Remove the two validated Risk/Market copies immediately before the P&L merge.
4. Check dashboard revision before preparing it, check again before publishing it, and keep one prepared cache.
5. Reuse the already combined MarketBook and build SearchCatalog lazily on first Quick Risk/Quick Market use.
6. Stop synchronously building the initial Risk and Aggregate P&L tables inside `build_layout()`.

Those changes attack the duplicate memory/CPU work without changing financial grain or business behaviour.

Then measure again before touching validation chains, sorting, or Portfolio grain.
