# THE FIX 2 — exact Rebirth V4.1 cold-start implementation guide

This is the prescriptive implementation version of `thefix.md`.

It is written against Rebirth V4.1 `main` around commit:

```text
cdf1e770f57eeb25e5cd5bd5a8620997ed1f90ee
```

The instructions below state exactly what to **keep**, **remove**, and **add or replace**. Apply the phases in order. Use one commit per phase, run the focused tests after every commit, and keep only changes that preserve the financial outputs while reducing elapsed time or peak RSS.

Do **not** merge the old `cold-start-change` branch wholesale. Do **not** apply all of `FIX10.md` in one commit. Do **not** apply Fix 4 as part of the core technical cold-start work.

---

# 0. Target result

After the core phases are complete:

1. Revision 1 is still produced by one guarded refresh writer.
2. The cold page reads only compact control metadata plus one defensive `dashboard_frame` copy.
3. The shared shell never reads the broad snapshot.
4. The same dashboard revision is prepared only once per process.
5. UI hydration cannot begin while the owned startup worker is still returning from refresh.
6. Validated Risk and Market frames are not copied immediately before their P&L merge.
7. The combined MarketBook is built once per revision.
8. `SearchCatalog` is built on the first Quick Risk/Quick Market use, not before revision 1 commits.
9. The initial Risk and Aggregate P&L tables are built by their normal callbacks, not synchronously inside `build_layout()`.
10. Connector batching and retry policy are changed only at explicit connector boundaries.

The following must remain unchanged:

- Risk values;
- dRisk values;
- P&L values and formulas;
- Risk-to-Market `many_to_one` validation;
- Open/Current quote uniqueness;
- Portfolio mapping;
- Reported Underlying mapping;
- RiskChecker `Age` and forced-date logic;
- Live/OFFICIAL routing;
- missing Market data remaining unavailable rather than becoming zero;
- atomic publication;
- last-good snapshot retention;
- expected-revision and reset-generation guards;
- official archive contracts.

---

# 1. Work rules and baseline

## KEEP

Keep the existing zero-click remount guard added by commit `99f7ceaa`. That fix prevents mounted buttons with `n_clicks == 0` from launching another manual refresh.

Keep:

```text
StartupCoordinator
_refresh_lock
_state_lock
_progress_lock
expected_revision
expected_reset_generation
_commit_snapshot
_commit_full_snapshot
_copy_snapshot for callers that explicitly request a broad result
connector validation
financial merge validation
```

## DO NOT DO

Do not:

- replace missing Market values with zero;
- remove `validate="many_to_one"`;
- remove quote uniqueness checks;
- remove source schema validation;
- remove last-good fallback;
- change Portfolio grain in the same pull request;
- remove row sorting before its consumers have been audited;
- add another app-wide cache around mutable DataFrames;
- use `future.result(timeout=...)` as a fake connector timeout;
- infer that the browser reconnect message means the Python process died.

## RECORD THE BASELINE

Before Step 2, record one real cold start with the same dates and connector scope:

```text
cold-start elapsed seconds
peak process RSS
revision reached
Risk source rows
combined_pl rows
dashboard_frame rows
Market rows
Market Open call count
Market Current call count
Risk call count
release seconds
search seconds
dashboard preparation seconds
initial layout seconds
revision-1-to-usable-page seconds
```

Also record whether a failure is:

```text
process exited or was OOM-killed
process remained alive but HTTP stopped responding
connector call stalled
revision 1 committed but browser was still synchronising
```

## BRANCH AND COMMIT ORDER

Use separate commits, preferably:

```text
coldstart/01-compact-control
coldstart/02-owned-ui-cache
coldstart/03-narrow-hydration
coldstart/04-completion-gate
coldstart/05-pl-input-ownership
coldstart/06-lazy-search
coldstart/07-defer-initial-tables
coldstart/08-bulk-market
coldstart/09-owned-validation
```

Do not squash them until the real-data measurements are complete.

---

# 2. Add the missing compact control metadata

The shared shell should use `ControlSnapshot`, but the compact model currently omits `refresh_reason`. Add that scalar field rather than reading a broad snapshot merely to display the last refresh reason.

## FILES

```text
rebirth/services/s01_snapshots.py
rebirth/app/s02_contracts.py
rebirth/services/s02_state.py
tests/s10_reads.py
```

## KEEP

Keep `ControlSnapshot` compact. It must continue to exclude:

```text
risk_checker inventory frame
combined_pl
market_frame
dashboard_frame
unmapped_frame
```

Keep the deep copy of `risk_status` and the dictionary copies of dates.

## ADD — `rebirth/services/s01_snapshots.py`

In `ControlSnapshot`, add `refresh_reason` immediately after `refreshed_at`.

Replace:

```python
@dataclass(frozen=True)
class ControlSnapshot:
    revision: int
    refreshed_at: datetime
    system_date: pd.Timestamp
```

with:

```python
@dataclass(frozen=True)
class ControlSnapshot:
    revision: int
    refreshed_at: datetime
    refresh_reason: str
    system_date: pd.Timestamp
```

Do not add any large DataFrame fields.

## ADD — `rebirth/app/s02_contracts.py`

In `ControlSnapshotProtocol`, add:

```python
    @property
    def refresh_reason(self) -> str: ...
```

Place it after `refreshed_at`.

Also add this property to `RefreshManagerProtocol`; it will be implemented in Step 5:

```python
    @property
    def writer_active(self) -> bool: ...
```

Place it near `health` and `progress`.

## ADD — `rebirth/services/s02_state.py`

Inside the `ControlSnapshot(...)` constructor in `control_snapshot`, add:

```python
            refresh_reason=committed.refresh_reason,
```

The start of the constructor should become:

```python
        return ControlSnapshot(
            revision=committed.revision,
            refreshed_at=committed.refreshed_at,
            refresh_reason=committed.refresh_reason,
            system_date=committed.system_date,
```

## TESTS

Add a test proving:

```python
control = manager.control_snapshot
assert control.refresh_reason == manager.snapshot.refresh_reason
assert not hasattr(control, "combined_pl")
assert not hasattr(control, "market_frame")
assert not hasattr(control, "dashboard_frame")
assert not hasattr(control, "unmapped_frame")
```

Also mutate `control.risk_status` and prove the committed manager frame is unchanged.

## RUN

```powershell
python -m pytest tests/s10_reads.py -q
```

## ACCEPTANCE GATE

This commit must not change any financial DataFrame or callback output. It adds one string to a compact metadata object only.

---

# 3. Give UI preparation an explicit ownership boundary

A `read_frame("dashboard_frame")` call already returns a private defensive copy. `prepare_risk_data()` currently copies that full frame again. Add an explicit ownership option, but preserve defensive behaviour for every ordinary caller.

At the same time, fix `_RiskDataCache.replace_frame()` so it checks the revision **before** doing expensive preparation and checks again before publication.

## FILES

```text
rebirth/ui/s02_aggregation.py
rebirth/pages/risk/s02_state.py
tests/s06_ui.py
tests/s10_reads.py
```

## KEEP

Keep all existing validation and transformation logic inside `prepare_risk_data()`.

Keep the default call defensive.

Keep the cache revision-keyed and keep clearing:

```text
_filtered
_rendered
_promotion_generations
```

when a newer revision is published.

## REPLACE — `rebirth/ui/s02_aggregation.py`

Replace the function header and first copy:

```python
def prepare_risk_data(data: pd.DataFrame) -> pd.DataFrame:
    """Validate and normalize a caller-owned DataFrame for the dashboard."""
    if not isinstance(data, pd.DataFrame):
        raise TypeError("data must be a pandas DataFrame")
    if data.empty:
        raise ValueError("data must contain at least one row")

    frame = data.copy()
```

with:

```python
def prepare_risk_data(
    data: pd.DataFrame,
    *,
    take_ownership: bool = False,
) -> pd.DataFrame:
    """Validate and normalize data, optionally consuming a private frame read."""
    if not isinstance(data, pd.DataFrame):
        raise TypeError("data must be a pandas DataFrame")
    if data.empty:
        raise ValueError("data must contain at least one row")
    if not isinstance(take_ownership, bool):
        raise TypeError("take_ownership must be boolean")

    frame = data if take_ownership else data.copy()
```

Do not change the rest of the function in this commit.

## REPLACE — `_RiskDataCache.current()` in `rebirth/pages/risk/s02_state.py`

Keep the existing fast path. Replace the final narrow read call:

```python
        dashboard = manager.read_frame("dashboard_frame")
        return self.replace_frame(dashboard.frame, dashboard.revision)
```

with:

```python
        dashboard = manager.read_frame("dashboard_frame")
        return self.replace_frame(
            dashboard.frame,
            dashboard.revision,
            take_ownership=True,
        )
```

## REMOVE — `_RiskDataCache.replace()`

After Step 4 updates the only cold-start caller, remove this broad-snapshot helper completely:

```python
    def replace(self, snapshot: RefreshSnapshotProtocol) -> pd.DataFrame:
        return self.replace_frame(snapshot.dashboard_frame, snapshot.revision)
```

Until Step 4 is applied, leave it temporarily so the intermediate commit is runnable. Delete it in the Step 4 commit and update any tests that call it.

## REPLACE — `_RiskDataCache.replace_frame()`

Replace the complete current method with:

```python
    def replace_frame(
        self,
        frame: pd.DataFrame,
        revision: int,
        *,
        take_ownership: bool = False,
    ) -> pd.DataFrame:
        """Prepare and publish one newer defensive dashboard-frame read."""
        selected_revision = int(revision)

        # A duplicate browser, route mount, or callback must not prepare a frame
        # that the cache has already published.
        with self._lock:
            if selected_revision <= self._revision:
                return self._frame

        prepared = prepare_risk_data(
            frame,
            take_ownership=take_ownership,
        )

        # Another request may have published while this request was preparing.
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

Do not hold `_lock` while running `prepare_risk_data()`. That calculation can be long and readers should retain the previous prepared revision during it.

The two revision checks are both required:

1. The first avoids duplicate work.
2. The second prevents an older prepared result from replacing a newer one.

## TESTS

Add tests proving:

1. Default `prepare_risk_data(frame)` leaves `frame` unchanged.
2. `prepare_risk_data(frame, take_ownership=True)` returns the same financial values.
3. A duplicate revision does not call `prepare_risk_data()`.
4. A stale revision does not call `prepare_risk_data()`.
5. If revision 8 publishes while revision 7 is preparing, revision 7 is discarded.
6. `current()` reads only `dashboard_frame` and passes `take_ownership=True`.

Use a monkeypatch counter around `prepare_risk_data()` for the duplicate/stale cases.

## RUN

```powershell
python -m pytest tests/s06_ui.py -q
python -m pytest tests/s10_reads.py -q
```

## ACCEPTANCE GATE

The default public behaviour remains defensive. Only a manager-owned defensive frame read may use `take_ownership=True`.

---

# 4. Replace broad startup reads and use one prepared dashboard cache

Current `build_app()` maintains a factory prepared cache and the Risk callbacks create a second `_RiskDataCache`. Current warm and cold Risk paths also read a broad `snapshot` that copies combined P&L, Market, dashboard, and unmapped frames.

This step creates one `_RiskDataCache` in the factory and passes it to every Risk callback group and the P&L prepared-frame loader.

## FILES

```text
rebirth/app/s07_factory.py
rebirth/pages/risk/s17_callbacks.py
rebirth/pages/risk/s15_refresh.py
rebirth/pages/risk/s16_view.py
rebirth/app/s02_contracts.py
tests/s10_reads.py
tests/s16_refreshshell.py
```

## KEEP

Keep:

- static DataFrame mode;
- `control_snapshot` for compact metadata;
- `read_frame("dashboard_frame")` for one defensive financial frame;
- one retry when control metadata and dashboard revision differ;
- current error shell behaviour;
- P&L using a prepared Risk frame only for its aggregate/filter UI.

## REMOVE — imports and duplicate caches in `rebirth/app/s07_factory.py`

Remove:

```python
from threading import Lock
```

Add:

```python
from rebirth.app.s02_contracts import (
    ControlSnapshotProtocol,
    RefreshManagerProtocol,
)
from rebirth.pages.risk.s02_state import _RiskDataCache
```

Remove the old one-line import of `RefreshManagerProtocol`.

Delete this complete block:

```python
    prepared_dashboard_lock = Lock()
    prepared_dashboard_revision = (
        int(initial_snapshot.revision) if initial_snapshot is not None else -1
    )
    prepared_dashboard_frame: pd.DataFrame | None = (
        risk_data if initial_snapshot is not None else None
    )
    risk_snapshot_lock = Lock()
    risk_snapshot_revision = prepared_dashboard_revision
    risk_snapshot_cache = initial_snapshot
```

## ADD — one revision-consistent narrow reader in `build_app()`

Place this helper after `history_repository` is created and before `initial_snapshot` is resolved:

```python
    def read_control_and_dashboard() -> tuple[
        ControlSnapshotProtocol,
        pd.DataFrame,
    ]:
        """Read one compact control view and one same-revision dashboard copy."""
        if refresh_manager is None:
            raise RuntimeError("A dashboard read requires a refresh manager")

        control = refresh_manager.control_snapshot
        dashboard = refresh_manager.read_frame("dashboard_frame")

        if int(control.revision) != int(dashboard.revision):
            control = refresh_manager.control_snapshot
            dashboard = refresh_manager.read_frame("dashboard_frame")

        if int(control.revision) != int(dashboard.revision):
            raise RuntimeError(
                "Could not read one consistent committed dashboard revision"
            )

        return control, dashboard.frame
```

Do not loop indefinitely. A second mismatch fails closed.

## REPLACE — warm initial snapshot handling

Replace:

```python
    initial_snapshot = None
    if refresh_manager is not None and refresh_manager.health.revision > 0:
        initial_snapshot = refresh_manager.snapshot
        risk_data = prepare_risk_data(initial_snapshot.dashboard_frame)
```

with:

```python
    initial_snapshot = None
    if refresh_manager is not None and refresh_manager.health.revision > 0:
        initial_snapshot, dashboard_frame = read_control_and_dashboard()
        risk_data = prepare_risk_data(
            dashboard_frame,
            take_ownership=True,
        )
```

Keep the existing cold-manager and static-data branches.

Immediately after `risk_data` is resolved, add the one shared cache:

```python
    risk_cache = _RiskDataCache(
        risk_data,
        int(initial_snapshot.revision) if initial_snapshot is not None else 0,
    )
```

## REPLACE — `prepared_committed_dashboard()`

Delete the old function body and its `nonlocal` factory cache.

Use:

```python
    def prepared_committed_dashboard() -> pd.DataFrame | None:
        """Return the one Risk-owned prepared frame for the current revision."""
        if refresh_manager is None:
            return risk_cache.current(None)
        try:
            if int(refresh_manager.health.revision) <= 0:
                return None
        except Exception:
            return None
        return risk_cache.current(refresh_manager)
```

Do not accept a supplied `revision` or `frame`. All manager-backed preparation must flow through the one cache.

## REPLACE — `current_cube_page()`

Remove:

```python
        nonlocal risk_snapshot_cache, risk_snapshot_revision
```

Remove the complete `risk_snapshot_lock` / `risk_snapshot_cache` block.

Use this manager-backed branch:

```python
        if refresh_manager is not None:
            try:
                revision = int(refresh_manager.health.revision)
                if revision > 0:
                    snapshot = refresh_manager.control_snapshot
                    prepared = risk_cache.current(refresh_manager)

                    if int(snapshot.revision) != int(risk_cache.revision):
                        snapshot = refresh_manager.control_snapshot
                        prepared = risk_cache.current(refresh_manager)

                    if int(snapshot.revision) != int(risk_cache.revision):
                        raise RuntimeError(
                            "Could not materialize one consistent Risk revision"
                        )

                    return build_layout(
                        prepared,
                        snapshot,
                        refresh_enabled=True,
                        stage_delays=stage_delays,
                        include_shared_refresh_shell=False,
                    )
```

Keep the existing exception shell and cold shell below it.

## KEEP — `current_shared_snapshot()`

Keep the existing compact call:

```python
return refresh_manager.control_snapshot
```

Do not replace it with `refresh_manager.snapshot`.

## REPLACE — Risk callback composition in `rebirth/pages/risk/s17_callbacks.py`

Remove:

```python
import pandas as pd
```

Change the contracts import to:

```python
from rebirth.app.s02_contracts import (
    ControlSnapshotProtocol,
    RefreshManagerProtocol,
    RefreshSnapshotProtocol,
)
```

Replace the function signature:

```python
def register_callbacks(
    app: Dash,
    refresh_manager: RefreshManagerProtocol | None,
    initial_snapshot: RefreshSnapshotProtocol | None,
    risk_data: pd.DataFrame,
```

with:

```python
def register_callbacks(
    app: Dash,
    refresh_manager: RefreshManagerProtocol | None,
    initial_snapshot: ControlSnapshotProtocol | RefreshSnapshotProtocol | None,
    cache: _RiskDataCache,
```

Delete this complete block from inside the function:

```python
    cache = _RiskDataCache(
        risk_data,
        initial_snapshot.revision if initial_snapshot is not None else 0,
    )
```

Keep every registration call and pass the supplied `cache` unchanged.

## REPLACE — factory registration call

Replace:

```python
    register_callbacks(
        app,
        refresh_manager,
        initial_snapshot,
        risk_data,
```

with:

```python
    register_callbacks(
        app,
        refresh_manager,
        initial_snapshot,
        risk_cache,
```

Keep `prepared_frame_loader=prepared_committed_dashboard` for P&L. It now uses the same Risk cache.

## REPLACE — cold Risk materialisation in `rebirth/pages/risk/s15_refresh.py`

Add `ControlSnapshotProtocol` to the contracts import.

Change the `initial_snapshot` and `materialize_initial_dashboard()` annotations to:

```python
ControlSnapshotProtocol | RefreshSnapshotProtocol | None
```

and:

```python
ControlSnapshotProtocol | RefreshSnapshotProtocol
```

Replace the first line inside the materialisation `try`:

```python
            prepared = cache.replace(snapshot)
```

with:

```python
            dashboard = refresh_manager.read_frame("dashboard_frame")
            if int(dashboard.revision) != int(snapshot.revision):
                snapshot = refresh_manager.control_snapshot
                dashboard = refresh_manager.read_frame("dashboard_frame")
            if int(dashboard.revision) != int(snapshot.revision):
                raise RuntimeError(
                    "Could not read one consistent initial dashboard revision"
                )
            prepared = cache.replace_frame(
                dashboard.frame,
                dashboard.revision,
                take_ownership=True,
            )
```

Now delete `_RiskDataCache.replace()` from Step 3.

Replace the cold success call:

```python
return materialize_initial_dashboard(refresh_manager.snapshot)
```

with:

```python
return materialize_initial_dashboard(refresh_manager.control_snapshot)
```

Replace the shared-shell success call:

```python
build_shared_refresh_shell(refresh_manager.snapshot, ...)
```

with:

```python
build_shared_refresh_shell(refresh_manager.control_snapshot, ...)
```

The shared shell must not read `dashboard_frame` or any other large frame.

## UPDATE — `rebirth/pages/risk/s16_view.py`

Change the contracts import to include `ControlSnapshotProtocol`.

Update these annotations:

```python
def build_risk_date_editor(
    snapshot: ControlSnapshotProtocol | RefreshSnapshotProtocol,
```

and:

```python
def build_layout(
    risk_data: pd.DataFrame,
    initial_snapshot: ControlSnapshotProtocol | RefreshSnapshotProtocol | None = None,
```

Do not change the financial layout logic in this commit.

## TESTS

Add tests that replace the broad `snapshot` property with a property that raises `AssertionError` and prove all of the following still work:

1. Warm app creation.
2. Cold Risk materialisation.
3. Shared-shell hydration.
4. Risk route remount.
5. P&L prepared-frame loading.

Also prove:

```text
one _RiskDataCache exists
factory has no risk_snapshot_cache
factory has no separate prepared_dashboard_frame
```

and that a control/dashboard revision race retries once.

## RUN

```powershell
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s16_refreshshell.py -q
python -m pytest tests/s09_plui.py -q
```

## ACCEPTANCE GATE

On a cold start, no manager-backed UI path may call the broad `snapshot` property before the user explicitly requests a workflow that needs a broad defensive result.

---

# 5. Do not report startup success before the writer has returned

Revision 1 is published before the refresh function has completely returned. The startup coordinator currently promotes any positive revision to `succeeded`, which can start heavy UI hydration while the startup worker is still alive and retaining final local frames.

Use both:

```text
revision > 0
and no active refresh writer
and no owned startup worker thread
```

as the success condition.

## FILES

```text
rebirth/services/s02_state.py
rebirth/app/s02_contracts.py
rebirth/app/s04_startup.py
rebirth/app/s05_progress.py
rebirth/pages/risk/s15_refresh.py
rebirth/ui/s04_components.py
tests/s12_startup.py
tests/s16_refreshshell.py
```

## KEEP

Keep:

- `RLock` as the one refresh writer gate;
- the non-destructive watchdog;
- one startup thread;
- `copy_result=False` in cold `_run()`;
- the external-writer follower;
- no attempt to kill an arbitrary connector thread;
- last-good data while a warm refresh runs.

## ADD — `writer_active` in `rebirth/services/s02_state.py`

Add this property near `health` and `progress`:

```python
    @property
    def writer_active(self) -> bool:
        """Return whether another thread currently owns the refresh transaction."""
        acquired = self._refresh_lock.acquire(blocking=False)
        if not acquired:
            return True
        self._refresh_lock.release()
        return False
```

This method is called by coordinator/status threads, not by the refresh owner itself. `RLock` is intentionally retained because `reset_refresh()` re-enters the normal refresh transaction.

The matching protocol property was added in Step 2.

## ADD — `StartupStatus.writer_active`

In `rebirth/app/s04_startup.py`, add:

```python
    writer_active: bool
```

to `StartupStatus`, after `worker_alive` or next to it.

Add this helper to `StartupCoordinator`:

```python
    def _writer_active(self) -> bool:
        try:
            return bool(self._manager.writer_active)
        except Exception:
            # Status must remain readable if an alternate test adapter fails.
            try:
                return bool(self._manager.progress.running)
            except Exception:
                return False
```

The real production manager must implement `writer_active`; the fallback is only to keep status reporting fail-soft for alternate adapters.

## REPLACE — top of `StartupCoordinator.start()`

Replace the initial positive-revision shortcut and worker check with:

```python
        with self._lock:
            worker_alive = self._worker is not None and self._worker.is_alive()
            writer_active = self._writer_active()

            if self._revision() > 0:
                if not worker_alive and not writer_active:
                    self._phase = "succeeded"
                    self._error = None
                elif self._phase == "idle":
                    self._phase = "running"
                return False

            if worker_alive or self._phase in {"running", "stalled", "succeeded"}:
                return False
```

Keep the retry rule, timer cancellation, attempt ID, worker construction, and logging below it.

Do not block startup on `writer_active` when revision is still zero; allowing the one coordinator worker to call `refresh()` and receive `RefreshInProgressError` is how it becomes the follower of an external writer.

## REPLACE — top of `schedule_start()`

Use the same order:

```python
        with self._lock:
            worker_alive = self._worker is not None and self._worker.is_alive()
            writer_active = self._writer_active()

            if self._revision() > 0:
                if not worker_alive and not writer_active:
                    self._phase = "succeeded"
                    self._error = None
                elif self._phase == "idle":
                    self._phase = "running"
                return False
```

Then retain the existing timer-alive and phase checks.

## REPLACE — `status()` success/watchdog logic

At the top of the locked block calculate:

```python
            worker_alive = self._worker is not None and self._worker.is_alive()
            writer_active = self._writer_active()
```

Replace the unconditional positive-revision success rule with:

```python
            if self._revision() > 0 and not worker_alive and not writer_active:
                self._phase = "succeeded"
                self._error = None
            elif (
                self._revision() > 0
                and (worker_alive or writer_active)
                and self._phase == "idle"
            ):
                self._phase = "running"
```

Change the watchdog condition from:

```python
and worker_alive
```

to:

```python
and (worker_alive or writer_active)
```

Return both fields:

```python
                worker_alive=worker_alive,
                writer_active=writer_active,
```

## REPLACE — `_follow_existing_writer()`

Replace the complete method with:

```python
    def _follow_existing_writer(self, attempt: int) -> None:
        while True:
            try:
                progress = self._manager.progress
                running = bool(progress.running)
                progress_error = progress.error
                writer_active = self._writer_active()
                revision = self._revision()
            except Exception as error:
                self._fail(attempt, error)
                return

            if revision > 0 and not running and not writer_active:
                self._succeed(attempt)
                return

            if not running and not writer_active:
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

Do not treat `revision > 0` by itself as completion.

## REPLACE — early path in `_run()`

Replace:

```python
            if self._revision() > 0:
                self._succeed(attempt)
                return
```

with:

```python
            if self._revision() > 0 and not self._writer_active():
                self._succeed(attempt)
                return
```

The normal `_succeed()` after `refresh()` returns remains unchanged.

In the `StaleRefreshError` and generic exception branches, only promote to success when:

```python
self._revision() > 0 and not self._writer_active()
```

Otherwise fail or follow the actual writer, rather than claiming completion from revision alone.

## ADD — progress payload fields

In `rebirth/app/s05_progress.py`, add to the base payload:

```python
        "writer_active": bool(
            getattr(refresh_manager, "writer_active", False)
            if refresh_manager is not None
            else False
        ),
```

When a startup coordinator is present, also add:

```python
            startup_worker_alive=startup.worker_alive,
            startup_writer_active=startup.writer_active,
```

Do not include DataFrames.

## ADD — one non-blocking hydration lock

In `rebirth/pages/risk/s15_refresh.py`, add:

```python
from threading import Lock
```

Inside `register_refresh_callbacks()`, before callback definitions, add:

```python
    startup_hydration_lock = Lock()
```

Replace the cold success branch with:

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

The shared shell no longer needs this lock because Step 4 made it metadata-only.

## REPLACE — cold-page interval only

In `rebirth/ui/s04_components.py`, change the `initial-load-trigger` interval:

```python
interval=500
```

to:

```python
interval=1_000
```

Keep `shared-refresh-bootstrap-interval` at 500 ms; after Step 4 it performs compact status/control work only.

## TESTS

Add tests proving:

1. A fake manager publishes revision 1 and then blocks before returning; startup remains `running` or watchdog `stalled`.
2. The same rule is exercised through `start()`, `schedule_start()`, and `status()`.
3. `_follow_existing_writer()` waits until both progress and the writer lock are inactive.
4. Success is reported immediately after the owned worker returns.
5. Two simultaneous cold hydration calls execute `materialize_initial_dashboard()` at most once concurrently.
6. The skipped caller can hydrate on a later poll.
7. A watchdog warning never starts a second writer.
8. A failed initial refresh remains retryable.
9. A failed warm refresh retains the previous revision.

## RUN

```powershell
python -m pytest tests/s12_startup.py -q
python -m pytest tests/s16_refreshshell.py -q
```

## ACCEPTANCE GATE

The browser must not start heavy Risk materialisation while the startup thread remains alive, even if revision 1 is already visible.

---

# 6. Remove the two unnecessary P&L input copies

`pandas.merge()` creates a new result. Copying already-validated Risk and Market immediately before the merge adds a large peak with no isolation benefit.

## FILES

```text
rebirth/domain/s03_calculations.py
tests/s04_market.py
tests/s07_integration.py
```

## KEEP

Keep:

```python
validate="many_to_one"
indicator="_market_merge"
```

Keep every P&L formula, market-availability rule, Gamma split, and input validation.

Do not make in-place assignments to the supplied `risk` or `market` frames.

## REPLACE

Replace:

```python
    if validated_risk is not None:
        if risk_source is not None:
            raise ValueError("validated_risk cannot be combined with a raw risk source")
        risk = validated_risk.copy()
```

with:

```python
    if validated_risk is not None:
        if risk_source is not None:
            raise ValueError("validated_risk cannot be combined with a raw risk source")
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

Keep the merge exactly:

```python
    result = risk.merge(
        market,
        on=spec.market_keys,
        how="left",
        validate="many_to_one",
        indicator="_market_merge",
    )
```

## TESTS

For an ordinary product and a Gamma product:

```python
expected_risk = risk.copy(deep=True)
expected_market = market.copy(deep=True)

result = get_product_pl(...)

pd.testing.assert_frame_equal(risk, expected_risk)
pd.testing.assert_frame_equal(market, expected_market)
assert result is not risk
assert result is not market
```

Also compare result values by full identity against the pre-change expected result.

## RUN

```powershell
python -m pytest tests/s04_market.py -q
python -m pytest tests/s07_integration.py -q
```

## ACCEPTANCE GATE

Risk, dRisk, P&L, Open, Current, Move, availability, and row identity must be identical to baseline.

---

# 7. Build the MarketBook once and build SearchCatalog lazily

Current full refresh builds `snapshot.market_frame`, then Search rebuilds/concatenates a second MarketBook projection and `SearchCatalog` deep-copies it. Search is also built before revision 1 can commit.

This step:

1. builds the combined MarketBook once;
2. gives Search an owned narrow projection of that frame;
3. removes eager Search from refresh and portfolio-only refresh;
4. builds one revision-keyed SearchCatalog on first use.

## FILES

```text
rebirth/domain/s10_search.py
rebirth/services/s06_refresh.py
rebirth/services/s02_state.py
rebirth/pages/risk/s08_quickrisk.py
rebirth/pages/risk/s09_quickmarket.py
rebirth/pages/risk/s14_workspacecallbacks.py
tests/s07_integration.py
tests/s10_reads.py
tests/s19_riskfilters.py
tests/s31_data.py
```

## KEEP

Keep:

- `snapshot.market_frame`;
- exact Quick Risk/Quick Market identity validation;
- bounded dropdowns;
- all eight manager Search methods;
- revision validation before caching;
- Search failure separate from the financial snapshot;
- defensive copying for direct public `SearchCatalog` construction.

## ADD — `rebirth/domain/s10_search.py`

Add this helper after `_market_catalog_frame()`:

```python
def _market_catalog_from_combined(
    market_frame: pd.DataFrame,
    market_date: pd.Timestamp,
) -> pd.DataFrame:
    """Project one already-combined MarketBook into the Search contract."""
    missing = [
        column for column in MARKET_RESULT_COLUMNS if column not in market_frame
    ]
    if missing:
        raise ValueError(f"combined MarketBook is missing Search columns: {missing}")

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

## REPLACE — `SearchCatalog.__init__()`

Add a defensive-default ownership flag:

```python
    def __init__(
        self,
        *,
        revision: int,
        risk_dates: Mapping[str, pd.Timestamp],
        market_date: pd.Timestamp,
        market_frame: pd.DataFrame,
        risk_pivot_frame: pd.DataFrame,
        copy_market_frame: bool = True,
    ) -> None:
```

Replace:

```python
        self._market_frame = market_frame.copy(deep=True)
```

with:

```python
        self._market_frame = (
            market_frame.copy(deep=True)
            if copy_market_frame
            else market_frame
        )
```

Direct callers keep the safe default. Only the builder below passes `False` after creating an owned projection.

## REPLACE — `build_search_catalog()`

Use this signature:

```python
def build_search_catalog(
    *,
    revision: int,
    risk_frames: Mapping[str, pd.DataFrame],
    market_frames: Mapping[str, pd.DataFrame] | None = None,
    market_frame: pd.DataFrame | None = None,
    risk_pivot_frame: pd.DataFrame | None = None,
    risk_dates: Mapping[str, pd.Timestamp],
    market_date: pd.Timestamp,
) -> SearchCatalog:
```

Keep the existing `risk_pivot_frame is None` fallback.

Replace the old unconditional Market build:

```python
    market_frame = _market_catalog_frame(market_frames, market_date)
```

with:

```python
    if market_frame is not None:
        owned_market_frame = _market_catalog_from_combined(
            market_frame,
            market_date,
        )
    elif market_frames is not None:
        owned_market_frame = _market_catalog_frame(
            market_frames,
            market_date,
        )
    else:
        raise ValueError("market_frame or market_frames is required")
```

Return:

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

## REPLACE — manager Search builder in `rebirth/services/s02_state.py`

Change `_build_snapshot_search_catalog()` so it accepts the already-combined frame:

```python
    @staticmethod
    def _build_snapshot_search_catalog(
        *,
        revision: int,
        risk_frames: Mapping[str, pd.DataFrame],
        market_frames: Mapping[str, pd.DataFrame] | None,
        market_frame: pd.DataFrame | None,
        dashboard: pd.DataFrame,
        risk_dates: Mapping[str, pd.Timestamp],
        market_date: pd.Timestamp,
        market_status: str,
    ) -> SearchCatalog:
        _require_market_status(market_status)
        return build_search_catalog(
            revision=revision,
            risk_frames=risk_frames,
            market_frames=market_frames,
            market_frame=market_frame,
            risk_pivot_frame=dashboard,
            risk_dates=risk_dates,
            market_date=market_date,
        )
```

## ADD — Search lock state in `rebirth/services/s06_refresh.py`

Change:

```python
from threading import RLock
```

to:

```python
from threading import Lock, RLock
```

In `__init__()`, immediately after `_progress_lock`, add:

```python
        self._search_catalog_lock = Lock()
```

Next to the existing Search member, add:

```python
        self._search_catalog: SearchCatalog | None = None
        self._search_catalog_revision = 0
```

Do not create another Search cache anywhere else.

## REPLACE — `_commit_full_snapshot()`

In `rebirth/services/s02_state.py`, remove this parameter:

```python
search_catalog: SearchCatalog,
```

Inside the atomic state publication, replace:

```python
            self._search_catalog = search_catalog
```

with:

```python
            self._search_catalog = None
            self._search_catalog_revision = 0
```

Every full financial or portfolio-mapping revision invalidates Search. Metadata-only commits that retain the same revision do not need to invalidate it.

## REMOVE — eager Search in `refresh_portfolios()`

Delete the complete block:

```python
                search_catalog = self._build_snapshot_search_catalog(
                    revision=revision,
                    risk_frames=risk_frames,
                    market_frames=market_frames,
                    dashboard=dashboard,
                    risk_dates=risk_dates,
                    market_date=market_date,
                    market_status=base_snapshot.market_status,
                )
```

Remove:

```python
search_catalog=search_catalog,
```

from `_commit_full_snapshot()`.

Keep all mapping, release, snapshot, and commit validation.

## REPLACE — full-refresh combined Market construction

Before constructing `RefreshSnapshot`, add:

```python
                combined_market = self._combined_market_frame(
                    next_market,
                    market_date,
                )
```

Replace the inline snapshot field:

```python
market_frame=self._combined_market_frame(next_market, market_date),
```

with:

```python
market_frame=combined_market,
```

## REMOVE — eager Search in full refresh

Delete:

```python
                search_started = time.monotonic()
                search_catalog = self._build_snapshot_search_catalog(
                    revision=revision,
                    risk_frames=next_risk,
                    market_frames=next_market,
                    dashboard=dashboard,
                    risk_dates=next_dates,
                    market_date=market_date,
                    market_status=expected_market_status,
                )
                stage_durations["search"] = time.monotonic() - search_started
```

Remove:

```python
search_catalog=search_catalog,
```

from the full `_commit_full_snapshot()` call.

Do not leave progress text claiming that Search is being built during refresh.

## ADD — lazy builder in `rebirth/services/s02_state.py`

Add:

```python
from time import monotonic
```

Then add this method beside the existing Search read methods:

```python
    def _current_search_catalog(self) -> SearchCatalog:
        """Build and cache Search once for the current committed revision."""
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

                    # These are stable committed references. The builder creates
                    # owned projections before retaining mutable DataFrames.
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
                    if current is None or int(current.revision) != revision:
                        # Release the Search lock, then retry against the newer
                        # committed revision.
                        continue
                    self._search_catalog = catalog
                    self._search_catalog_revision = revision
                    return catalog
```

Do not hold `_refresh_lock` while constructing Search.

Do not mark the financial snapshot failed when Search construction fails.

## REPLACE — all eight Search methods

In each method below, delete the direct `_search_catalog` state read and `None` check:

```text
combine_udl_options
market_udl_options
resolve_history_identity
search_market_udl_options
search_combine_udl_options
pivot_market_exact
pivot_combined
pivot_combined_hierarchy
```

Start each method with:

```python
        catalog = self._current_search_catalog()
```

Keep the existing call on `catalog` unchanged.

Every one of these methods must be safe as the first Search action after a refresh.

## ADD — visible Search status in Quick Risk

In `rebirth/pages/risk/s08_quickrisk.py`, directly below the existing selector-help span, add:

```python
                                    html.Span(
                                        "",
                                        id="quick-risk-catalog-status",
                                        className="quick-search-selector-help",
                                        role="status",
                                    ),
```

Do not reuse `quick-search-data-status`; that belongs to the Open in Data handoff.

## ADD — visible Search status in Quick Market

In `rebirth/pages/risk/s09_quickmarket.py`, directly after the identity dropdown, add:

```python
                                    html.Span(
                                        "",
                                        id="quick-market-catalog-status",
                                        className="quick-search-selector-help",
                                        role="status",
                                    ),
```

Do not reuse `quick-market-data-status`.

## REPLACE — Quick Risk options callback outputs

In `rebirth/pages/risk/s14_workspacecallbacks.py`, add a third output:

```python
Output("quick-risk-catalog-status", "children"),
```

When the tab is inactive return:

```python
return no_update, no_update, no_update
```

On failure use:

```python
            except (
                AttributeError,
                LookupError,
                TypeError,
                ValueError,
                RuntimeError,
            ) as error:
                app.logger.warning(
                    "Quick Risk catalog unavailable: %s",
                    type(error).__name__,
                )
                return (
                    no_update,
                    no_update,
                    "Search index is temporarily unavailable. Reopen this tab to retry.",
                )
```

Do not expose raw exception text to the browser.

On success, add `""` as the third return value.

For example:

```python
            if selected in values:
                return options, no_update, ""
            return options, (options[0]["value"] if options else None), ""
```

## REPLACE — Quick Market options callback outputs

Add:

```python
Output("quick-market-catalog-status", "children"),
```

Use the same three-value return pattern and a Quick Market-specific warning log.

## TESTS

Prove:

1. Full refresh reaches revision 1 without calling `build_search_catalog()`.
2. `refresh_portfolios()` also commits without building Search.
3. A previous catalog is invalidated by a new full revision.
4. The first Search call builds once.
5. Two simultaneous first Search calls build once.
6. Opening Quick Market after Quick Risk reuses the same catalog.
7. Each of the eight public methods can be the first Search action.
8. A refresh during Search build prevents stale catalog publication.
9. Search failure leaves revision 1 healthy and retries later.
10. Quick Risk and Quick Market display a local generic failure message.
11. `_combined_market_frame()` is called once per full financial refresh.
12. Quick Market values and ordering match the baseline.
13. Direct `SearchCatalog(...)` construction still copies its input by default.

## RUN

```powershell
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s19_riskfilters.py -q
python -m pytest tests/s31_data.py -q
```

## ACCEPTANCE GATE

Revision 1 must commit before any Search index is built. First-use Search may be slower, but the main Risk page must become usable sooner and all later Search actions in the same revision must reuse the catalog.

---

# 8. Remove eager initial Risk and Aggregate P&L table construction

`build_layout()` currently constructs both full tables synchronously, even though normal callbacks already own those outputs.

## FILES

```text
rebirth/pages/risk/s16_view.py
rebirth/pages/risk/s07_explorer.py
rebirth/pages/risk/s14_workspacecallbacks.py
assets/s01_base.css or the appropriate Risk CSS file
tests/s06_ui.py
tests/s19_riskfilters.py
tests/s34_riskpivot.py
```

## KEEP

Keep:

- default filter values;
- `default_filtered` if required to seed controls;
- initial Risk type;
- initial IR family;
- `initial_open_rows`;
- all stores and component IDs;
- the normal Risk reducer callback;
- the normal Aggregate P&L callback;
- initial-call behaviour of those callbacks.

Do not add another interval to trigger table construction.

## REMOVE — imports in `rebirth/pages/risk/s16_view.py`

Remove:

```python
build_aggregate_pl_table,
```

from `rebirth.ui.s04_components` imports.

Remove:

```python
from .s06_explorertables import build_risk_table
```

Only remove imports that become unused.

## REMOVE — eager builder calls

Delete the complete `initial_risk_table = build_risk_table(...)` block.

Delete:

```python
    initial_aggregate_table = build_aggregate_pl_table(
        default_filtered,
        DEFAULT_VIEW_DIMENSION,
        [],
    )
```

Keep the calculations needed for `initial_open_rows` and control values.

## REPLACE — Aggregate P&L initial child

Replace:

```python
html.Div(
    initial_aggregate_table,
    id="aggregate-pl-grid",
)
```

with:

```python
html.Div(
    "Preparing current view…",
    id="aggregate-pl-grid",
    className="table-placeholder",
)
```

Keep the existing `dcc.Loading` wrapper and output ID.

## REPLACE — Risk initial child

Replace:

```python
html.Div(
    initial_risk_table,
    id="risk-grid",
    className="risk-grid",
)
```

with:

```python
html.Div(
    "Preparing current view…",
    id="risk-grid",
    className="risk-grid table-placeholder",
)
```

Keep `alt-risk-grid`, `detail-panel`, stores, and all callback IDs unchanged.

## ADD — small placeholder CSS

Add:

```css
.table-placeholder {
  min-height: 96px;
  display: grid;
  place-items: center;
  color: var(--muted-text, #667085);
  font-size: 0.9rem;
}
```

Do not add an animated loader beyond the existing `dcc.Loading` boundary.

## VERIFY CALLBACKS

The Aggregate callback in `s14_workspacecallbacks.py` must remain initial-call enabled. Do not add `prevent_initial_call=True`.

The Risk reducer in `s07_explorer.py` must remain initial-call enabled. Do not add `prevent_initial_call=True`.

More than one initial callback execution may occur while stores settle. Step 3's revision precheck and existing render cache prevent duplicate frame preparation; do not claim exactly one callback invocation.

## TESTS

Prove:

1. `build_layout()` does not call `build_risk_table()`.
2. `build_layout()` does not call `build_aggregate_pl_table()`.
3. Both stable output IDs exist with placeholders.
4. Both callbacks are initial-call enabled.
5. The first callback graph replaces both placeholders.
6. Base filter defaults remain unchanged.
7. Initial Risk open rows remain unchanged.
8. Aggregate totals equal the baseline.
9. Chevron actions still work.

## RUN

```powershell
python -m pytest tests/s06_ui.py -q
python -m pytest tests/s19_riskfilters.py -q
python -m pytest tests/s34_riskpivot.py -q
```

## ACCEPTANCE GATE

The full page shell must return without synchronously building either large table. The callbacks must populate both automatically after mount.

---

# 9. Add explicit optional bulk Market hooks

The current fallback makes one Open and one Current call per Underlying. Threading can reduce latency, but it does not reduce call count. A source that supports a list of Underlyings should make one bulk Open call and one bulk Current call.

This step changes connector routing only. It does not alter validation or financial calculations.

## FILES

```text
rebirth/domain/s02_products.py
rebirth/services/s06_refresh.py
rebirth/services/s05_sources.py or the private production connector module
tests/s07_integration.py
```

## KEEP

Keep required scalar hooks:

```text
risk
market_open
market_status
```

They remain the fallback for sources without bulk APIs.

Keep all canonical Market validators after the connector returns.

Keep `_reject_unrequested_market_underlyings()`.

## ADD — bulk protocol in `rebirth/domain/s02_products.py`

After `ProductMarketConnector`, add:

```python
class ProductBulkMarketConnector(Protocol):
    """One source-bound Market connector for an ordered Underlying scope."""

    def __call__(
        self,
        source_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame: ...
```

Append optional fields to `ProductConnectorAdapter`:

```python
    market_open_bulk: ProductBulkMarketConnector | None = None
    market_status_bulk: ProductBulkMarketConnector | None = None
```

Append them after the three existing required fields so existing three-positional-argument constructions remain valid.

Update the adapter docstring to state that bulk hooks receive the complete ordered tuple once.

## ADD — constructor validation in `RiskRefreshManager.__init__()`

Keep the required-hook validation. Add:

```python
            invalid_optional_hooks = [
                hook
                for hook in ("market_open_bulk", "market_status_bulk")
                if getattr(adapter, hook) is not None
                and not callable(getattr(adapter, hook))
            ]
            if invalid_optional_hooks:
                raise TypeError(
                    f"connector adapter for {source_type!r} has non-callable "
                    f"optional hooks: {invalid_optional_hooks}"
                )
```

## ADD — one bulk loader in `rebirth/services/s06_refresh.py`

Add after `_load_market_frames()`:

```python
    def _load_bulk_market_frame(
        self,
        spec: ProductSpec,
        underlyings: tuple[str, ...],
        *,
        connector: Callable[..., pd.DataFrame],
        source_date: pd.Timestamp,
        market_status: str,
        stage: str,
        label: str,
    ) -> pd.DataFrame:
        connector_name = _callable_name(connector)
        self._progress_activity(
            connector_name,
            stage,
            source_type=spec.source_type,
            product_index=1,
            product_total=1,
            message=(
                f"Loading {label} for {len(underlyings):,} Underlyings "
                "in one bulk call."
            ),
        )
        frame = connector(
            source_date,
            underlyings,
            market_status=market_status,
        )
        if not isinstance(frame, pd.DataFrame):
            kind = "market Open" if stage == "market_open" else "current market"
            raise TypeError(f"bulk {kind} connector must return a pandas DataFrame")
        return frame
```

This first bulk patch deliberately does not add another retry loop. Native connector timeouts and retry classification are handled below.

## REPLACE — Open routing

At the start of `_load_product_market_open()`, after resolving `adapter` and `selected_status`, add:

```python
        if adapter is not None and adapter.market_open_bulk is not None:
            return self._load_bulk_market_frame(
                spec,
                underlyings,
                connector=adapter.market_open_bulk,
                source_date=open_date,
                market_status=selected_status,
                stage="market_open",
                label="Open",
            )
```

Keep the existing scalar `_load_market_frames()` fallback below it.

## REPLACE — Current routing

At the start of `_load_product_market_status()`, after resolving `adapter` and `selected_status`, add:

```python
        if adapter is not None and adapter.market_status_bulk is not None:
            return self._load_bulk_market_frame(
                spec,
                underlyings,
                connector=adapter.market_status_bulk,
                source_date=market_date,
                market_status=selected_status,
                stage="market_status",
                label=selected_status,
            )
```

Keep the scalar fallback.

## REPLACE — logical call counters

For Open, replace the unconditional Underlying count with:

```python
                    adapter = self._connector_adapters.get(source_type)
                    market_open_calls += (
                        1
                        if adapter is not None
                        and adapter.market_open_bulk is not None
                        else len(requested_underlyings)
                    )
```

Use the matching rule for Current.

## ADD — real production connector wrappers

For each selected source type, define explicit functions shaped like:

```python
def market_open_bulk(
    open_date: pd.Timestamp,
    underlyings: tuple[str, ...],
    *,
    market_status: str,
) -> pd.DataFrame:
    raw = SITE_OPEN_API(
        source_date=open_date,
        underlyings=list(underlyings),
        market_status=market_status,
        timeout=SITE_TIMEOUT_SECONDS,
    )
    return normalize_site_open(raw)
```

and:

```python
def market_status_bulk(
    market_date: pd.Timestamp,
    underlyings: tuple[str, ...],
    *,
    market_status: str,
) -> pd.DataFrame:
    raw = SITE_CURRENT_API(
        source_date=market_date,
        underlyings=list(underlyings),
        market_status=market_status,
        timeout=SITE_TIMEOUT_SECONDS,
    )
    return normalize_site_current(raw)
```

Use the timeout argument actually supported by the client. Do not paste a made-up keyword.

Register the hooks only for a source whose service really supports bulk.

## RETRY POLICY — REMOVE THE BLANKET POLICY ONLY AFTER NATIVE TIMEOUTS EXIST

Current manager-wide policy retries nearly every exception four times under a 20-thread fan-out. Do not retain that as the final production policy.

After every real connector has a native I/O timeout:

1. Remove universal retry of arbitrary `Exception`.
2. Retry only explicit transient classes, normally `TimeoutError`, `ConnectionError`, and the site's documented throttling exception.
3. Do not retry authentication, permission, schema, `TypeError`, `ValueError`, or programming failures.
4. Use connector-owned exponential backoff with jitter and a total deadline.
5. Set concurrency to the service's approved limit, not a universal 20.

A safe connector-owned shape is:

```python
for attempt in range(max_attempts):
    try:
        return site_call(..., timeout=timeout_seconds)
    except TRANSIENT_EXCEPTIONS:
        if attempt + 1 == max_attempts:
            raise
        sleep(backoff_with_jitter(attempt))
```

Do not catch `BaseException`.

Do not retry data validation after download.

## TESTS

Prove:

1. Bulk Open receives the complete ordered tuple once.
2. Bulk Current receives the complete ordered tuple once.
3. A scalar adapter still receives one call per Underlying.
4. Bulk output still passes canonical Open/Current validation.
5. Unexpected Underlyings fail.
6. Duplicate quote keys fail.
7. A correctly shaped empty bulk frame means unavailable Market data.
8. A failed bulk leg publishes no partial revision.
9. Call metrics report one logical bulk call.
10. Values and keys equal the concatenated scalar baseline.

## RUN

```powershell
python -m pytest tests/s07_integration.py -q
python -m pytest -q
```

## ACCEPTANCE GATE

Enable one source type first. Compare exact sorted keys, tenor orders, Open, Current, Move, P&L, and availability before enabling a second source.

---

# 10. Optional measured phase — one owned connector-validation frame

Apply this only after Steps 2–9 are stable and measurements still show large copy peaks during connector validation.

Current validation can copy the same full frame in `_load_frame()`, `_enforce_product()`, `_require_nonblank()`, `_coerce_numeric()`, tenor validation, and final projection.

The correct ownership rule is:

```text
one copy at the connector boundary
then mutate only that private working frame
public/shared helpers remain defensive by default
```

## FILES

```text
rebirth/domain/s03_calculations.py
rebirth/domain/s07_governance.py
tests/s03_adapters.py
tests/s04_market.py
tests/s07_integration.py
tests/s14_reporting.py
```

## KEEP

Keep this boundary unchanged:

```python
return frame.copy()
```

inside `_load_frame()`.

That is the single source-ownership copy.

Keep every numeric, nonblank, product identity, tenor authority, duplicate, and merge check.

## REPLACE — `_require_nonblank()`

Use:

```python
def _require_nonblank(
    frame: pd.DataFrame,
    columns: list[str],
    label: str,
    *,
    owned: bool = False,
) -> pd.DataFrame:
    result = frame if owned else frame.copy()
    # keep the current validation loop unchanged
```

## REPLACE — `_coerce_numeric()`

Use:

```python
def _coerce_numeric(
    frame: pd.DataFrame,
    columns: list[str],
    label: str,
    *,
    owned: bool = False,
) -> pd.DataFrame:
    result = frame if owned else frame.copy()
    # keep the current validation loop unchanged
```

## REPLACE — private pipeline helpers

In `_enforce_product()` replace:

```python
result = frame.copy()
```

with:

```python
result = frame
```

In `_validate_market_tenor_orders()` make the same replacement.

These helpers are private pipeline functions. Do not expose them as public mutation helpers.

## ADD — `owned=True` in connector validation chains

After `_load_frame()` has returned the private working frame, pass `owned=True` to `_require_nonblank()` and `_coerce_numeric()` in:

```text
get_product_risk
get_product_market_open
get_product_market_status
```

Also pass it for optional Credit numeric measures.

Do not pass `owned=True` from governance/public callers such as `merge_config()`.

## REPLACE — final owned projections

For connector validator return paths, replace:

```python
return frame[columns].copy()
```

with:

```python
return frame.loc[:, columns]
```

Only do this where `_load_frame()` established ownership.

## TESTS

For Risk, Open, and Current:

1. Deep-copy the connector source.
2. Run the validator.
3. Prove the source is unchanged.
4. Prove the output schema and dtypes are unchanged.

Also prove `merge_config()` does not mutate its supplied P&L frame.

## RUN

```powershell
python -m pytest tests/s03_adapters.py -q
python -m pytest tests/s04_market.py -q
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s14_reporting.py -q
```

## ACCEPTANCE GATE

Keep this phase only if real peak RSS or elapsed time improves. Revert one helper at a time if an ownership test fails.

---

# 11. Changes deliberately deferred

## DO NOT REMOVE THE P&L SORT YET

Keep the ordinary `get_product_pl()` sort for the core patch.

Removing it may reduce allocation, but it changes an observable row-order contract and can affect:

- archive reproducibility;
- `keep="first"` downstream behaviour;
- external comparisons;
- Gamma row adjacency;
- option order;
- debugging.

Only remove it in a separate measured PR after auditing every consumer and changing tests to compare by complete financial identity.

## DO NOT REMOVE PORTFOLIO FROM RISK IN THE CORE PATCH

Keep Portfolio in:

```text
raw Risk
get_product_pl
combined_pl
Portfolio mapping
Reported Underlying mapping
unmapped diagnostics
P&L sending
Stock
official risk.parquet
current dashboard_frame
Quick Risk
Risk filters
```

Fix 4 can produce the largest row-count reduction, but it is a business/data-grain change. It removes Portfolio from Risk analytics and controls. It requires separate approval and reconciliation.

Only open that PR after the technical phases are measured. Its non-negotiable flow is:

```text
Risk at Portfolio grain
-> Market join
-> P&L
-> Portfolio mapping
-> Reported Underlying
-> promotions
-> combined_pl remains detailed
-> mapped analytical dashboard collapses across Portfolio
```

Official history and P&L must retain the detailed Portfolio projection.

Do not use `drop_duplicates()` as a collapse. Additive values must be summed with `min_count=1`; quote/governance values must be proven consistent before carrying one value.

## DO NOT REMOVE SEARCH SOURCE CACHES

Keep `_risk_frames`, `_market_frames`, `_pl_frames`, and overlays. They are used by partial refreshes, portfolio-only refreshes, detail flows, and lazy Search.

## DO NOT REMOVE THE WATCHDOG

The watchdog does not hold financial frames. It reports a stalled call and prevents a second writer. The real fix for a hung connector is a native I/O timeout.

## DO NOT TREAT HEALTH/PROGRESS POLLING AS THE MAIN MEMORY PROBLEM

Their payloads are scalar metadata. The important fixes are narrow reads, one prepared cache, completion gating, lazy Search, deferred table construction, and connector batching.

---

# 12. Final test gate

After every focused suite passes, run:

```powershell
python -m pytest -q
python -m ruff check .
python -m ruff format --check .
git diff --check
```

## FINANCIAL RECONCILIATION

Capture a baseline snapshot before the change set and compare after:

```python
import pandas as pd
import pytest

identity = [
    column
    for column in (
        "Source Type",
        "Risk Type",
        "Risk Greek",
        "Split",
        "Underlying",
        "Tenor Swap",
        "Tenor Option",
        "Portfolio",
        "Region",
    )
    if column in before.combined_pl.columns
]

before_pl = before.combined_pl.sort_values(
    identity,
    kind="stable",
    na_position="last",
).reset_index(drop=True)

after_pl = after.combined_pl.sort_values(
    identity,
    kind="stable",
    na_position="last",
).reset_index(drop=True)

pd.testing.assert_frame_equal(before_pl, after_pl)

for column in ("Risk", "dRisk", "PL"):
    assert after.dashboard_frame[column].sum(min_count=1) == pytest.approx(
        before.dashboard_frame[column].sum(min_count=1)
    )
```

Also compare:

```text
Market keys
Open
Current
Move
Market Available
Market Data Status
risk_dates
market_date
market_status
Portfolio Mapped
Reported Underlying
promotion values
unmapped rows
```

## COLD-START OPERATIONAL CHECK

Verify:

1. App shell paints before connector I/O.
2. Exactly one startup writer runs.
3. `/healthz` and `/progressz` remain responsive.
4. Revision 1 commits.
5. Startup does not report success while its thread is alive.
6. No broad snapshot is copied for Risk hydration or shared shell.
7. One dashboard preparation occurs per revision.
8. Search has not built at revision-1 publication.
9. Risk and Aggregate placeholders are automatically replaced.
10. First Quick Risk/Quick Market use builds Search once.
11. Later Search uses the same catalog.
12. A failed refresh retains the last good revision.
13. Clear Cache still advances reset generation and performs one guarded full refresh.
14. P&L and Stock retain Portfolio behaviour.
15. Peak RSS and total elapsed time are recorded against baseline.

---

# 13. Rollback map

Each phase has no stored-data migration and should be independently reversible.

```text
Step 2 rollback: remove ControlSnapshot.refresh_reason additions.
Step 3 rollback: restore unconditional prepare_risk_data copy and old cache method.
Step 4 rollback: restore factory caches and broad snapshot reads.
Step 5 rollback: restore revision-only startup success and remove hydration lock.
Step 6 rollback: restore the two validated input copies.
Step 7 rollback: restore eager Search build and SearchCatalog commit parameter.
Step 8 rollback: restore eager layout table builders.
Step 9 rollback: omit optional bulk hooks and use scalar fallback.
Step 10 rollback: restore copies one validator helper at a time.
```

Do not roll back atomic commit, validation, or last-good retention to recover performance.

---

# 14. Recommended stopping point

The recommended first production trial consists of Steps 2 through 8:

```text
compact control metadata
owned UI preparation
one prepared dashboard cache
narrow hydration
true completion gate
single-flight hydration
remove two P&L copies
one combined MarketBook
lazy SearchCatalog
deferred initial tables
```

Measure that version before applying connector batching or the owned validator chain.

If the process still dies **before revision 1**, focus next on connector call volume, native timeouts, P&L merge peaks, release intermediates, and Market batching.

If the process dies **immediately after revision 1**, focus on duplicate dashboard preparation, UI tenor resolution, component-tree construction, and response serialization.

If the process remains alive but the browser disconnects, measure CPU time in preparation and table callbacks and confirm the proxy can still reach `/healthz` and `/progressz`.

If the core technical patch succeeds but the Portfolio-grain dashboard remains too large, open the separate Fix 4 business-grain PR. Do not mix that semantic change into the cold-start infrastructure patch.
