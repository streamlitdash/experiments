# Step 4 — Troubleshoot SOG and Portfolio P&L

Use this guide when the P&L page has no SOG choices, no Portfolio choices, or
empty editor tables. Diagnose the first empty or invalid boundary in order; do
not fill missing market P&L with zero merely to make a sender table appear.

There is no hardcoded `No SOG found` message in the repository. That wording is
the dropdown's no-options state: the effective frame was empty, no current
snapshot existed, or the callback that builds the choices failed.

The two editor sections do not have separate inbound feeds. Both are views of
the same current governed P&L frame:

- SOG P&L selects rows by `SignoffGroup` and allows Portfolio to be changed to
  another governed Portfolio inside that SOG.
- Portfolio P&L selects one `Portfolio` and keeps Portfolio locked.
- Send All passes independent copies of the same governed payload to the SOG
  and Portfolio outbound functions.

Therefore, when both selectors are empty, troubleshoot the shared current P&L
path before changing either editor callback.

#### 1. Separate current P&L from historical P&L

These are independent paths:

| P&L feature | Authoritative input |
|---|---|
| SOG P&L, Portfolio P&L and Send All | current committed `RiskRefreshManager.pl_snapshot.combined_pl` |
| P&L Overview and inline History | completed historical archive queried by `SQLPLHistoryRepository` |
| Validate P&L | completed daily `risk.parquet` and `colossus.parquet` leaves |
| Saved adjustments | `PL_ADJUSTMENT_PATH/<Market Date>/...csv` |

Missing historical Parquet can make current/month/year history unavailable, but
it does not create the SOG or Portfolio dropdown values. Conversely, a valid
current SOG table does not prove that historical files are present.

#### 2. Follow the current Risk-to-P&L path

The checked-in path is:

```text
ProductConnectorAdapter.risk(risk_date)
  -> get_product_risk
  -> validated position rows at raw Underlying + tenor + Portfolio grain

ProductConnectorAdapter.market_open(checker_date, Underlying)
ProductConnectorAdapter.market_status(market_date, Underlying)
  -> get_product_market_open + get_product_market_status
  -> one Open/Current quote per raw Underlying + declared tenor grain
  -> _merge_validated_market_legs

validated Risk many_to_one validated MarketBook
  -> get_product_pl
  -> one product P&L frame; missing market remains PL=NaN

all product P&L frames
  -> _release_pl_views
  -> Portfolio config joins by exact Portfolio
  -> Reported Underlying mapping
  -> promotion thresholds
  -> atomically committed RefreshSnapshot.combined_pl

RiskRefreshManager.pl_snapshot
  -> pages/pnl/s05_sendcallbacks.py::refresh_effective_query
  -> pages/pnl/s02_editor.py::_effective_rows
  -> discard Portfolio Mapped=False rows
  -> apply the committed P&L-page filters
  -> map Risk Type + Risk Greek to ConcertoField
  -> sum to Market Date + Portfolio + ConcertoField
  -> optionally overlay saved adjustments
  -> populate SOG and Portfolio choices
  -> validate again immediately before sending
```

The important ownership rules are:

- Risk supplies `Portfolio`; it does not supply Activity, SignoffGroup,
  Category or Sub Category.
- `get_portfolio_config()` supplies those fields later by exact Portfolio
  mapping.
- Market does not contain Portfolio. Many Portfolio positions intentionally
  join to one market quote with `many_to_one` validation.
- P&L is calculated before portfolio and reporting metadata are attached.
- The sender excludes unmapped Portfolios. It does not invent a SignoffGroup.
- Historical Colossus/Predict rows are not merged into the current send frame.

#### 3. Check that a current snapshot actually committed

Open the lowercase health endpoints using the same application prefix as the
Cube URL:

```text
/healthz
/progressz
```

For JupyterHub or a proxied Plotly deployment, append `healthz` or `progressz`
to the deployed application prefix rather than to the host root.

A ready current P&L path has:

```text
healthz.status = "ok" (or "degraded" only because an older error is retained)
healthz.revision >= 1
progressz.running = false
progressz.startup_phase = "succeeded" (or "idle" for an already-warm manager)
```

If `revision` is `0`, no `pl_snapshot` exists and empty SOG/Portfolio selectors
are only a downstream symptom. Read `progressz.function_name`, `stage`,
`source_type`, `underlying`, `message` and `error`, then fix the first source or
validator failure. Do not begin with the P&L page callbacks.

On a cold-start failure, the current manager requires every registered
`PRODUCT_SPECS` Source Type to produce a valid Risk frame. Risk frames cannot be
empty. Open and Current frames may be empty with the correct columns, but their
positions will then have unavailable P&L and cannot enter a governed send.

The checked-in `rebirth/services/s05_sources.py` is still an explicit fake-CSV
boundary. If real values were pasted into those CSVs without replacing the
loader, `_require_fake_notice()` rejects them because reporting identities must
contain `FAKE_REPLACE_ME`. For a real deployment, replace the connector bodies
or registered adapters; do not remove that fixture warning and continue using
the fake loader.

#### 4. Reproduce one complete refresh outside the UI

Stop the local app first so the notebook does not start a second set of real
connector calls. Start Jupyter in the repository root and run:

```python
from rebirth.services.s05_sources import build_production_refresh_manager

manager = build_production_refresh_manager()

try:
    snapshot = manager.refresh(
        force_risk=True,
        force_pl=True,
        reason="SOG and Portfolio P&L diagnostic",
    )
except Exception:
    print(manager.progress)
    raise

print("revision:", snapshot.revision)
print("market date:", snapshot.market_date)
print("combined P&L rows:", len(snapshot.combined_pl))
print("dashboard rows:", len(snapshot.dashboard_frame))
print("unmapped rows:", len(snapshot.unmapped_frame))
```

If this raises, the traceback is the actual upstream problem. Fix it before
testing the editor. If it succeeds, keep `snapshot` for the remaining checks so
the connectors are not called again.

#### 5. Prove that mapped rows have finite current P&L

Run this against the diagnostic snapshot:

```python
import numpy as np
import pandas as pd

combined = snapshot.combined_pl.copy()
mapped = combined["Portfolio Mapped"].eq(True)
pl_number = pd.to_numeric(combined["PL"], errors="coerce")
finite_pl = pl_number.notna() & np.isfinite(pl_number)

print(
    {
        "combined_rows": len(combined),
        "mapped_rows": int(mapped.sum()),
        "unmapped_rows": int((~mapped).sum()),
        "mapped_nonfinite_pl": int((mapped & ~finite_pl).sum()),
        "signoff_groups": int(
            combined.loc[mapped, "SignoffGroup"].nunique(dropna=True)
        ),
        "portfolios": int(combined.loc[mapped, "Portfolio"].nunique()),
    }
)

problem_columns = [
    column
    for column in (
        "Source Type",
        "Risk Type",
        "Risk Greek",
        "Underlying",
        "Tenor Swap",
        "Tenor Option",
        "Portfolio",
        "Open",
        "Current",
        "Market Data Status",
        "PL",
    )
    if column in combined
]
print(
    combined.loc[mapped & ~finite_pl, problem_columns]
    .head(25)
    .to_string(index=False)
)
```

Interpret the result:

- `mapped_rows == 0`: the Risk Portfolio values do not match portfolio
  governance, or the applied page filter removed every governed Portfolio.
- `mapped_nonfinite_pl > 0`: at least one send candidate has no valid current
  P&L. `build_pl_send_base()` intentionally rejects the whole candidate rather
  than sending a partial or fabricated total.
- `signoff_groups == 0`: no mapped governance supplied a usable
  `SignoffGroup`.
- all four values are positive/zero as expected: continue to the Concerto and
  editor-state checks.

For non-finite P&L, use `Market Data Status` to fix the feed:

| Status/problem | Fix |
|---|---|
| `No matching market row` | Make Risk and market use the same raw Underlying and declared tenor labels. |
| `Missing Open` | Return the exact opening quote key for the checker date. |
| `Missing Current (Live/OFFICIAL)` | Return the exact current quote key for Market Date and selected status. |
| percentage product has `Open == 0` | Supply the real non-zero opening level. |
| duplicate quote/position error before commit | Fix the duplicate at its real Risk or market key; do not hide it with `drop_duplicates()`. |

Open and Current are quote-grain data. Do not add Portfolio to market keys, and
do not replace a missing quote with zero.

#### 6. Validate exact Portfolio governance

Risk Portfolio values are matched exactly and case-sensitively, after
surrounding whitespace is normalized, to the Portfolio connector. The
connector must return:

```text
Portfolio, Product, Activity, SignoffGroup, Category
```

`Sub Category` is optional and defaults to `Unspecified`. Portfolio must be
unique and nonblank; the other supplied values must be nonblank; Product must
be exactly `XVA` or `Hedges` after case normalization.

List the unmapped Risk values:

```python
print(
    combined.loc[~mapped, "Portfolio"]
    .drop_duplicates()
    .sort_values()
    .head(100)
    .to_string(index=False)
)
```

Fix either the Risk connector's Portfolio identity or the portfolio mapping
source. Do not add Activity or SignoffGroup columns to Risk or derive a SOG from
the book name. After correcting governance, use Refresh Portfolios when only
the mapping changed; use Refresh Risk when the Risk Portfolio values changed.

#### 7. Validate the Concerto mapping and build the exact send base

`CONCERTO_MAPPING_PATH` defaults to `data/s08_concerto.csv`. It must contain
exactly these columns in this order:

```text
Risk Type,Risk Greek,ConcertoField
```

All fields must be nonblank. Each Risk Type + Risk Greek pair must occur once,
and each ConcertoField must belong to one pair only. Every mapped send candidate
must have a governed pair.

Run the same public validators used by the page:

```python
import os
from pathlib import Path

from rebirth.app.s01_settings import resolve_data_path
from rebirth.domain.s01_schema import PORTFOLIO_METADATA_COLUMNS
from rebirth.domain.s08_pnl import (
    build_pl_send_base,
    load_plsend_mapping,
    load_portfolio_governance,
)

project_root = Path.cwd().resolve()
mapping_path = resolve_data_path(
    os.getenv("CONCERTO_MAPPING_PATH"),
    Path("data/s08_concerto.csv"),
    root=project_root,
)
mapping = load_plsend_mapping(mapping_path)

governance_source = (
    combined.loc[
        mapped,
        ["Portfolio", *PORTFOLIO_METADATA_COLUMNS],
    ]
    .drop_duplicates()
    .reset_index(drop=True)
)
governance = load_portfolio_governance(governance_source)
base = build_pl_send_base(combined, mapping, governance)

print("mapping:", mapping_path)
print("governed base rows:", len(base))
print("SOG choices:", sorted(base["SignoffGroup"].unique().tolist()))
print("Portfolio choices:", sorted(base["Portfolio"].unique().tolist())[:25])
print(base.head(20).to_string(index=False))
```

The base schema is:

```text
Market Date, Risk Type, Risk Greek, Portfolio, SignoffGroup,
ConcertoField, PL, Adjustment
```

It is unique at `Market Date + Portfolio + ConcertoField` after aggregating
position-level P&L. `Adjustment` is `False` for calculated rows. The validator
also proves that SignoffGroup matches Portfolio governance and ConcertoField
matches the Risk Type + Risk Greek mapping.

If this cell raises, its message identifies the missing pair, contradictory
SOG/Concerto value, non-finite PL, or duplicate governed identity. Fix that
source; do not weaken `validate_pl_send_rows()`.

#### 8. Remove page filters as a cause

The editors use only the committed saved-view state. Draft selector changes do
not apply until the user clicks Apply.

1. Select `Base Review`.
2. Click Apply.
3. Confirm the applied Activity values exist in the current mapping. The Base
   view resolves Activity 1, 2 and 3 (including the checked-in aliases).
4. Temporarily leave the remaining four selectors blank and keep Exclude off.
5. Open SOG P&L or Portfolio P&L. An odd summary-click count means open; closed
   sections deliberately do no row work.
6. After any Risk refresh changes the revision, close and reopen the editor if
   it says that the snapshot or Market Date changed.

If the notebook base is populated but the page choices are empty, read the
server traceback for the callback that owns
`pl-send-effective-query-store.data`. Also read the visible row status:

```text
Choose a SignoffGroup to load rows.
No governed Portfolio belongs to <scope>.
No PL rows are available for <scope>.
Could not prepare <scope> rows: <validation error>.
```

The callback query must carry the current snapshot revision, Market Date,
applied filter scope and whether adjustments are included. It deliberately
rejects a stale browser store rather than mixing two revisions.

#### 9. Test saved adjustments separately

First leave `Show adjustments` off. If calculated rows load with it off but fail
with it on, the current Risk/market path is healthy and the problem is under
`PL_ADJUSTMENT_PATH` (default `adjustments`).

Saved rows must use the same eight base columns, have `Adjustment=True`, match
the snapshot Market Date, obey the Portfolio and Concerto mappings, and be
unique by `Market Date + Portfolio + ConcertoField`. Use the error shown by
`LocalCsvAdjustmentRepository.load()` to correct or remove only the bad dated
Portfolio file; do not delete the entire adjustment directory.

#### 10. Distinguish loading from outbound sending

The checked-in functions in `rebirth/services/s05_sources.py` are safe fixture
boundaries:

```python
send_sog_pl(frame)
send_portfolio_pl(frame)
```

They intentionally raise `RuntimeError` for every real send. This does not
cause empty dropdowns; they are called only after rows have loaded and the user
clicks Send.

After both editor tables load correctly, replace the bodies with authorized
site-owned send calls. The current outbound payload has exactly:

```text
Risk Type, Risk Greek, Portfolio, SignoffGroup, ConcertoField, PL, Adjustment
```

The sender must accept a pandas DataFrame, return normally only after the
destination accepts it, and raise on failure. Send All tries both destinations
with defensive copies and reports complete, partial, or total failure. Market
Date is validated before sending but is not in the current outbound payload; if
the site API requires it, change that contract deliberately with tests rather
than guessing it inside the sender.

#### 11. Use this shortest decision tree

```text
health revision == 0
  -> fix the first Risk/Open/Current/PL refresh failure

revision >= 1 but mapped_rows == 0
  -> fix exact Portfolio mapping and SignoffGroup governance

mapped_nonfinite_pl > 0
  -> fix missing/mismatched Open or Current quote keys

build_pl_send_base raises
  -> fix Concerto mapping, governance, PL or governed-key duplication

base has rows but browser has no choices
  -> apply Base Review, remove filters, reopen the section, inspect callback log

rows load only when adjustments are off
  -> fix the dated adjustment file

rows load but Send fails
  -> replace the intentionally disabled SOG/Portfolio outbound functions
```

#### 12. Run the focused validation gate

After fixing the responsible source, run:

```powershell
python -m pytest tests/s05_pl.py -q
python -m pytest tests/s09_plui.py -q
python -m pytest tests/s07_integration.py -q
python -m pytest -q
git diff --check
```

The minimum real-data acceptance check is:

```python
assert snapshot.revision >= 1
assert mapped.any()
assert not (mapped & ~finite_pl).any()
assert not base.empty
assert base["SignoffGroup"].nunique() > 0
assert base["Portfolio"].nunique() > 0
```

Do not test the real outbound functions until the governed base passes and the
recipient has authorized a test destination.
