# Fix 4 — Remove Portfolio from the Risk analytical layer

Use this change when Portfolio-level Risk rows make the final refresh,
Quick Risk catalogue, Risk hierarchy, or Dash responses unnecessarily large.
The change keeps Portfolio wherever it is financially required and removes it
only after P&L and governance have been calculated.

This is not a connector change. Risk must still arrive at Portfolio grain.
Portfolio must still be used to attach Product, Activity, SignoffGroup,
Category, and Sub Category.

For 370 Credit underlyings, 10 tenors, and about 70 Portfolios, the detailed
position frame can contain roughly 259,000 rows per Greek. If the remaining
reporting dimensions agree, the Risk analytical frame can approach 3,700 rows.
Different Activities, Products, categories, or other reporting identities
correctly remain separate.

The browser component named dimension-filter-values-store contains only small
filter selections. It does not contain the DataFrame. Removing the Portfolio
selector cleans up the Risk workflow, but the major performance improvement
comes from collapsing Portfolio before publishing dashboard_frame.

#### 1. Preserve the correct data-grain boundaries

Use this ownership rule throughout the fix:

| Area | Portfolio |
|---|---|
| Raw Risk connector | Keep |
| Product P&L and combined_pl | Keep |
| Portfolio mapping and Reported Underlying mapping | Keep |
| Unmapped-book diagnostics | Keep and display |
| Stock and Stock history | Keep |
| SOG P&L, Portfolio P&L, adjustments, and sending | Keep |
| Official risk.parquet history | Keep |
| Risk dashboard_frame | Remove after aggregation |
| Risk filter and View-by controls | Remove |
| Quick Risk pivot/filter dimensions | Remove |
| Quick Market | Unchanged; it never has Portfolio |

Do not drop Portfolio from the connector, before get_product_pl(), or before
_merge_validated_config(). Doing so would lose the mapping authority and would
break Portfolio P&L.

The required flow is:

~~~text
Risk at Portfolio grain
  -> Open/Current join
  -> P&L
  -> Portfolio mapping
  -> Reported Underlying
  -> baseline promotions
  -> combined_pl remains detailed
  -> mapped rows collapse across Portfolio
  -> Portfolio-free dashboard_frame
  -> Risk Explorer / Aggregate P&L / Quick Risk
~~~

#### 2. Retain one Portfolio-grain projection for official history

File: rebirth/domain/s07_governance.py

Rename the current to_dashboard_frame() function to:

~~~python
def to_portfolio_archive_frame(frame: pd.DataFrame) -> pd.DataFrame:
    """Return the validated Portfolio-grain projection used by history."""
~~~

Keep the existing body unchanged, including:

- Vol Score normalization;
- the required Portfolio input column;
- the current ordered column list;
- Portfolio in the returned frame.

This preserves the current official risk.parquet structure without duplicating
the projection logic.

Then add the new analytical to_dashboard_frame() immediately after that
function:

~~~python
def _sum_available(values: pd.Series):
    """Sum values while preserving an entirely missing group as missing."""

    return values.sum(min_count=1)


def to_dashboard_frame(frame: pd.DataFrame) -> pd.DataFrame:
    """Publish one mapped, Portfolio-free analytical frame."""

    positions = to_portfolio_archive_frame(frame)
    credit_columns = [
        column for column in CREDIT_MEASURE_COLUMNS if column in positions
    ]
    additive_columns = [RISK, DRISK, PL, *credit_columns]
    carried_quote_columns = [
        OPEN,
        CURRENT,
        MARKET_AVAILABLE,
        MARKET_DATA_STATUS,
    ]
    carried_governance_columns = [
        DISPLAY_BUCKET,
        PROMOTION_REASON,
        RISK_THRESHOLD,
        DRISK_THRESHOLD,
        PL_THRESHOLD,
    ]
    carried_columns = [
        *carried_quote_columns,
        *carried_governance_columns,
    ]
    excluded_from_identity = {
        PORTFOLIO,
        MARKET_MOVE,
        VOL_SCORE,
        PROMOTION_SCORE,
        *additive_columns,
        *carried_columns,
    }
    group_columns = [
        column
        for column in positions.columns
        if column not in excluded_from_identity
    ]

    # Rows being collapsed must repeat one quote/governance state. Never hide a
    # connector or merge error by selecting a contradictory value.
    quote_counts = (
        positions.groupby(
            group_columns,
            dropna=False,
            sort=False,
            observed=True,
        )[carried_columns]
        .nunique(dropna=False)
    )
    inconsistent = quote_counts.gt(1).any(axis=1)
    if inconsistent.any():
        examples = (
            quote_counts.loc[inconsistent]
            .head(5)
            .reset_index()
            .to_dict("records")
        )
        raise ValueError(
            "Portfolio rows contain contradictory carried values at the "
            f"dashboard aggregation grain: {examples}"
        )

    aggregations: dict[str, object] = {
        **{column: _sum_available for column in additive_columns},
        VOL_SCORE: "max",
        PROMOTION_SCORE: "max",
        **{column: "first" for column in carried_columns},
    }
    dashboard = (
        positions.groupby(
            group_columns,
            as_index=False,
            dropna=False,
            sort=False,
            observed=True,
        )
        .agg(aggregations)
    )
    dashboard[MARKET_MOVE] = dashboard[CURRENT] - dashboard[OPEN]

    output_columns = [
        column for column in positions.columns if column != PORTFOLIO
    ]
    return dashboard.loc[:, output_columns].copy()
~~~

The aggregation rules are deliberate:

- Risk, dRisk, P&L, SP01, PSP01, PM01, PM01P, Theta, JTD, and any other
  registered Credit measure are additive.
- An all-missing dRisk, P&L, or Credit measure stays missing. It must not become
  zero.
- Open and Current are quote values, not position values. They are retained
  once only after proving that repeated Portfolio rows agree.
- Move is recalculated as Current minus Open.
- Market Available and Market Data Status are retained. This preserves special
  states such as Commodity market disabled.
- Promotion Score and Vol Score are ranking signals and use max. They must
  never be summed.
- Display Bucket, Promotion Reason, and thresholds are validated as repeated
  governance values and retained once.
- Every remaining field is part of the analytical identity. Therefore rows
  with different Activity, Product, SignoffGroup, Category, Sub Category,
  Split, raw Underlying, reported Underlying, tenor, Group, or Region do not
  collapse incorrectly. Contradictory quote or carried-governance values fail
  before publication.

Do not use drop_duplicates(). It would discard Risk/P&L. Do not sum Open or
Current because that would weight a quote by Portfolio count.

The existing _validate_dashboard_release() already validates a Portfolio-free
release and can remain after to_dashboard_frame() in
RiskRefreshManager._release_pl_views().

Strengthen _validate_dashboard_release() in the same file:

- raise if Portfolio is present in the released frame;
- include Region in the validated grain when the column exists;
- reject duplicate rows at the complete released analytical grain;
- continue requiring every Portfolio Mapped value to be True.

This catches an incomplete collapse before SearchCatalog or the browser sees
the revision.

#### 3. Keep official history at Portfolio grain

The current archive writes snapshot.dashboard_frame. That must change in the
same patch, otherwise risk.parquet will lose Portfolio.

File: rebirth/history/s02_contracts.py

In OfficialSnapshot, replace:

~~~python
dashboard_frame: pd.DataFrame
~~~

with:

~~~python
combined_pl: pd.DataFrame
~~~

File: rebirth/history/s03_io.py

Import:

~~~python
from rebirth.domain.s01_schema import PORTFOLIO_MAPPED_COLUMN
from rebirth.domain.s07_governance import to_portfolio_archive_frame
~~~

Inside archive_official_snapshot(), replace:

~~~python
risk = validate_risk_archive_frame(snapshot.dashboard_frame)
~~~

with:

~~~python
detailed = snapshot.combined_pl
if PORTFOLIO_MAPPED_COLUMN not in detailed:
    raise RiskArchiveValidationError(
        "committed P&L is missing 'Portfolio Mapped'"
    )
mapped = detailed.loc[detailed[PORTFOLIO_MAPPED_COLUMN].eq(True)].copy()
risk = validate_risk_archive_frame(
    to_portfolio_archive_frame(mapped)
)
~~~

This retains the previous mapped-only archive policy and the previous ordered
columns, including Portfolio. It does not archive unmapped positions into
official P&L history.

Update the module and protocol docstrings so they say that official history
stores the mapped Portfolio-grain governed P&L projection, not dashboard_frame.

Do not change the Parquet schema version for this fix because the archived
Portfolio-grain schema remains the same.

#### 4. Add a Risk-only four-field registry

The current five-field registry is also used by Stock, P&L, and the persistent
saved-view repository. Keep that registry unchanged. Add a Risk-only projection
instead; this is smaller and avoids breaking Portfolio workflows elsewhere.

File: rebirth/ui/s01_constants.py

Keep PORTFOLIO_UI_FIELD, FILTER_DIMENSION_ORDER, FILTER_DIMENSION_FIELDS,
FILTER_COLUMNS, and DIMENSION_FILTER_IDS unchanged as the five-field
position-grain contract.

Change VIEW_DIMENSION_FIELDS so the Risk analytical table no longer offers the
synthetic Portfolio field:

~~~python
VIEW_DIMENSION_FIELDS = tuple(
    field
    for field in PORTFOLIO_FIELDS
    if "view_dimension" in field.roles
)
~~~

Then add:

~~~python
RISK_FILTER_DIMENSION_FIELDS = tuple(
    field
    for field in FILTER_DIMENSION_FIELDS
    if field.key != "portfolio"
)
RISK_FILTER_COLUMNS = [
    field.key for field in RISK_FILTER_DIMENSION_FIELDS
]
RISK_DIMENSION_FILTER_IDS = {
    field.key: field.dash_filter_id
    for field in RISK_FILTER_DIMENSION_FIELDS
}
~~~

Export all three new names in __all__.

The resulting Risk controls are:

~~~text
Activity
Signoff Group
Category
Sub Category
~~~

Risk View-by retains Product, Activity, Signoff Group, Category, and
Sub Category. Portfolio is absent. Default View-by remains Activity.

Stock and P&L continue using the unchanged FILTER_DIMENSION_FIELDS tuple:

~~~text
Activity
Signoff Group
Portfolio
Category
Sub Category
~~~

#### 5. Switch every Risk consumer to the Risk-only registry

Do not change Stock or P&L imports. Change only the Risk analytical path:

| File | Replace with Risk-only authority |
|---|---|
| rebirth/pages/risk/s01_common.py | saved-view fields, filter IDs, reporting_filter_map(), quick_risk_filter_map() |
| rebirth/pages/risk/s03_defaults.py | default value tuple and payload |
| rebirth/pages/risk/s04_handoff.py | handoff length check and field zip |
| rebirth/pages/risk/s07_explorer.py | option/value callback zips |
| rebirth/pages/risk/s11_promotion.py | PromotionBasis filter-key validation |
| rebirth/pages/risk/s12_promotecallbacks.py | default arrays and filter map |
| rebirth/pages/risk/s16_view.py | rendered controls and committed Store |
| rebirth/pages/risk/s17_callbacks.py | callback component ID list |

Use:

~~~python
from rebirth.ui.s01_constants import (
    RISK_DIMENSION_FILTER_IDS,
    RISK_FILTER_COLUMNS,
    RISK_FILTER_DIMENSION_FIELDS,
)
~~~

Import only the names each module needs. Keep positional arrays and strict zip
calls aligned to RISK_FILTER_DIMENSION_FIELDS.

File: rebirth/ui/s02_aggregation.py

prepare_risk_data() and apply_filters() are Risk analytical functions. Change
their FILTER_COLUMNS use to RISK_FILTER_COLUMNS. The prepared Risk frame must
not synthesize an Unspecified portfolio column, and a programmatic Portfolio
filter must be rejected as unknown.

VIEW_DIMENSIONS automatically loses Portfolio because
VIEW_DIMENSION_FIELDS was changed above.

Leave these existing lines unchanged:

~~~python
# rebirth/pages/stock/s01_data.py
STOCK_FILTER_FIELDS = FILTER_DIMENSION_FIELDS

# rebirth/pages/pnl/s01_common.py
PL_FILTER_FIELDS = FILTER_DIMENSION_FIELDS
~~~

The SavedFilterViewRepository in rebirth/app/s07_factory.py also stays on the
five-field FILTER_DIMENSION_FIELDS schema.

#### 6. Let Risk use a four-field projection of shared saved views

Risk, Stock, and P&L share the named-view catalogue, but Risk now displays four
of the five stored fields. Adapt the generic callback boundary instead of
changing or deleting saved files.

File: rebirth/ui/s03_filters.py

Add these helpers near the saved-view conversion helpers:

~~~python
def _project_saved_view(
    view: SavedFilterView,
    controls: SavedFilterViewControls,
) -> SavedFilterView:
    """Return only the fields owned by one page."""

    return SavedFilterView(
        identifier=view.identifier,
        scope=view.scope,
        name=view.name,
        filters={
            field.key: tuple(view.filters.get(field.key, ()))
            for field in controls.fields
        },
        exclude_selected=view.exclude_selected,
    )


def _repository_filter_payload(
    repository: SavedFilterViewRepository,
    controls: SavedFilterViewControls,
    filter_values,
    *,
    existing: SavedFilterView | None = None,
):
    """Expand one page selection to the repository's superset schema."""

    payload = {
        key: list(existing.filters[key])
        if existing is not None
        else []
        for key in repository.filter_keys
    }
    payload.update(selected_filter_payload(controls, filter_values))
    return payload
~~~

In register_saved_filter_view_callbacks(), change the exact-key guard to a
subset guard:

~~~python
field_keys = tuple(field.key for field in controls.fields)
unknown = set(field_keys) - set(repository.filter_keys)
if unknown:
    raise ValueError(
        f"Saved view UI fields are not supported by the repository: "
        f"{sorted(unknown)}"
    )
~~~

Apply these rules inside the existing callbacks:

1. Saving a new Risk view expands its four visible filters to the five-key
   repository contract, with Portfolio empty.
2. Updating an existing Risk view first loads the full stored view, then
   replaces only the four visible keys. Its hidden Portfolio selection is
   preserved for Stock and P&L.
3. Loading a saved view for Risk passes
   _project_saved_view(repository.get(...), controls) into
   saved_view_apply_request().
4. Stock and P&L continue to use all five fields and therefore behave exactly
   as before.

For example, the update branch should follow this shape:

~~~python
existing = repository.get(
    controls.scope,
    selected_identifier,
)
filters = _repository_filter_payload(
    repository,
    controls,
    filter_values,
    existing=existing,
)
view = repository.update(
    controls.scope,
    selected_identifier,
    filters,
    exclude_selected=exclude_selected,
)
~~~

After this change, the Risk committed-state and
dimension-filter-values-store contain four filters. Stock and P&L retain five.
The stores still contain selections only, never financial rows.

#### 7. Move P&L filter options off dashboard_frame

The P&L page currently uses the prepared Risk dashboard only to populate its
five filter dropdowns. Once dashboard_frame becomes Portfolio-free, that would
silently empty the P&L Portfolio options.

File: rebirth/pages/pnl/s01_common.py

Import PORTFOLIO_MAPPED_COLUMN and UNSPECIFIED_VALUE, then add:

~~~python
def prepare_pl_filter_frame(combined_pl: pd.DataFrame) -> pd.DataFrame:
    """Return mapped, deduplicated P&L filter combinations."""

    if not isinstance(combined_pl, pd.DataFrame):
        raise TypeError("combined P&L must be a pandas DataFrame")
    required = [
        PORTFOLIO_MAPPED_COLUMN,
        *(field.external_name for field in PL_FILTER_FIELDS),
    ]
    missing = [column for column in required if column not in combined_pl]
    if missing:
        raise ValueError(
            f"combined P&L is missing P&L filter columns: {missing}"
        )

    mapped = combined_pl.loc[
        combined_pl[PORTFOLIO_MAPPED_COLUMN].eq(True),
        [field.external_name for field in PL_FILTER_FIELDS],
    ].copy()
    mapped.columns = [field.key for field in PL_FILTER_FIELDS]
    for field in PL_FILTER_FIELDS:
        mapped[field.key] = (
            mapped[field.key]
            .fillna(UNSPECIFIED_VALUE)
            .astype(str)
            .str.strip()
            .replace("", UNSPECIFIED_VALUE)
        )
    return mapped.drop_duplicates(ignore_index=True)
~~~

File: rebirth/app/s07_factory.py

Add a separate revision-cached prepared_committed_pl_filters() helper. It must:

1. read only read_frame("combined_pl") for a new committed revision;
2. call prepare_pl_filter_frame();
3. cache the small deduplicated result by revision;
4. return defensive copies only where the caller can mutate them.

Use that helper in pnl_page_body() for the initial filter frame and pass it to
register_pnl_callbacks(). Do not pass prepared_committed_dashboard to P&L.

File: rebirth/pages/pnl/s08_aggregate.py

Rename prepared_frame_loader to pl_filter_frame_loader for clarity. Make
current_filter_frame() use that loader. Its fallback path should read
combined_pl and call prepare_pl_filter_frame(), not read dashboard_frame or
call prepare_risk_data().

Update the forwarding parameter in rebirth/pages/pnl/__init__.py.

This helper supplies dropdown options only. P&L Overview history remains owned
by the history repository, while the SOG/Portfolio editors continue using
snapshot.combined_pl. Do not change their calculations.

#### 8. Remove Portfolio from Quick Risk

The Quick Risk catalogue is built during the final refresh stage. It must stop
requiring Portfolio in the same commit as dashboard_frame, or cold start will
fail while constructing SearchCatalog.

File: rebirth/domain/s10_search.py

Remove PORTFOLIO_COLUMN from the schema import and delete the local PORTFOLIO
constant.

Replace the relevant registries with:

~~~python
PIVOT_INDEX_COLUMNS = (
    SOURCE_TYPE,
    RISK_TYPE,
    RISK_GREEK,
    REPORTED_UNDERLYING,
    UNDERLYING,
    *TENOR_COLUMNS,
    *PORTFOLIO_METADATA_COLUMNS,
)
GOVERNANCE_COLUMNS = tuple(PORTFOLIO_METADATA_COLUMNS)
RISK_ONLY_INDEX_COLUMNS = (
    REPORTED_UNDERLYING,
    *GOVERNANCE_COLUMNS,
)

QUICK_RISK_FILTER_COLUMNS = (
    SPLIT,
    *(
        field.external_name
        for field in PORTFOLIO_FIELDS
        if "filter_dimension" in field.roles
    ),
)
~~~

In _risk_pivot_catalog_frame(), the required columns become:

~~~python
required = [
    SOURCE_TYPE,
    RISK_TYPE,
    RISK_GREEK,
    UNDERLYING,
    *RISK_PIVOT_VALUE_COLUMNS,
]
~~~

In build_search_catalog(), remove:

~~~python
fallback[PORTFOLIO] = UNSPECIFIED
~~~

File: rebirth/pages/risk/s08_quickrisk.py

Remove:

~~~python
("Portfolio", "Portfolio"),
~~~

from _QUICK_SEARCH_IDENTITY_OPTIONS.

Quick Risk still aggregates Risk, dRisk, and P&L using the selected reported or
raw Underlying and tenor hierarchy. Quick Market remains based on unique raw
quotes and must not be changed.

#### 9. Remove the Portfolio-only unmapped filter

Unmapped rows have no governed Activity, SignoffGroup, Category, or
Sub Category. They should remain a complete diagnostic inventory.

File: rebirth/pages/risk/s02_state.py

Delete filter_unmapped_portfolios() and remove it from __all__.

File: rebirth/pages/risk/s07_explorer.py

For the callback that renders unmapped-books-grid:

- remove dimension-filter-values-store and
  risk-filter-exclude-applied-store from its Inputs;
- remove dimension_values and exclude_value from the function arguments;
- delete the Portfolio-index lookup;
- delete the filter_unmapped_portfolios() call.

Continue loading refresh_manager.read_frame("unmapped_frame") lazily only while
the disclosure is open. Continue displaying Portfolio in
build_unmapped_books_table(); it is the identity needed to repair the mapping.

#### 10. Avoid copying combined_pl when mounting the Risk page

dashboard_frame becomes much smaller, but the current warm Risk route can still
call refresh_manager.snapshot. That property defensively copies combined_pl,
market_frame, dashboard_frame, unmapped_frame, and other large data even though
the Risk layout only needs control metadata plus dashboard_frame.

Use the existing targeted manager reads:

~~~text
refresh_manager.control_snapshot
refresh_manager.read_frame("dashboard_frame")
~~~

Update the warm-start and Risk-layout paths in:

- rebirth/app/s07_factory.py;
- rebirth/pages/risk/s15_refresh.py;
- the relevant control/layout protocol annotations in
  rebirth/app/s02_contracts.py and rebirth/pages/risk/s16_view.py.

The flow should be:

~~~python
control = refresh_manager.control_snapshot
dashboard = refresh_manager.read_frame("dashboard_frame")
prepared = prepared_committed_dashboard(
    revision=dashboard.revision,
    frame=dashboard.frame,
)
layout = build_layout(
    prepared,
    control,
    refresh_enabled=True,
    # existing remaining arguments
)
~~~

Use one committed revision consistently; reject or retry if the control and
frame revisions do not agree. Type layout inputs against ControlSnapshotProtocol
where only compact control fields are read.

Keep the New Trades drilldown's lazy read_frame("combined_pl") because that
feature genuinely needs detailed position rows. Do not put combined_pl back
into normal Risk mount or table callbacks.

This step removes a second large position-frame copy from Risk navigation. It
does not change the defensive-copy behavior of explicit refresh APIs.

#### 11. Update Risk wording and defaults

File: rebirth/pages/risk/s01_common.py

Update RISK_FILTER_NOTE so it does not mention Portfolio:

~~~python
RISK_FILTER_NOTE = (
    "Include mode uses OR within one populated filter and AND across "
    "populated filters. Exclude mode removes rows matching any selected "
    "value. Leave a filter blank for all values; Risk selections remain "
    "independent from Stock and P&L."
)
~~~

The existing defaults in rebirth/pages/risk/s03_defaults.py automatically
align to the new four-field Risk registry. Keep Activity 1, Activity 2, and
Activity 3 as the initial Base selection. The other three Risk filters remain
empty.

Search the complete Risk path for stale Portfolio dependencies:

~~~powershell
rg -n "Portfolio|portfolio" rebirth/pages/risk rebirth/domain/s10_search.py rebirth/ui/s01_constants.py
~~~

Portfolio is still expected in unmapped-table copy and in explanatory comments
about mapping. It must not remain as a Risk dropdown, View-by option, Quick Risk
dimension, or applied Risk filter.

#### 12. Add focused regression tests

Update tests/s14_reporting.py or add an equivalent focused governance test:

1. Two mapped rows differing only by Portfolio collapse into one dashboard row.
2. Risk, dRisk, P&L, and Credit measures equal the two-row sums.
3. Open and Current are retained once and Move remains Current minus Open.
4. Vol Score becomes the maximum, not the sum.
5. Portfolio is absent from dashboard_frame.
6. Portfolio remains present with both original rows in combined_pl.
7. An all-missing P&L group remains missing.
8. Contradictory Open/Current values at the collapse grain raise ValueError.
9. Rows with different Activity or Product do not collapse.

Update tests/s19_riskfilters.py:

- Risk filters are Activity, Signoff Group, Category, and Sub Category.
- Portfolio is not rendered as a Risk filter.
- Portfolio is not offered as a Risk View-by dimension.
- the Risk committed store has four selections;
- Aggregate P&L cannot group by Portfolio;
- the unmapped diagnostic still displays all Portfolio rows;
- Quick Risk rejects Portfolio as a pivot/filter dimension and returns the
  correct cross-Portfolio sum.

Update tests/s23_savedviews.py:

- Risk controls use four fields;
- Stock and P&L controls use five fields including Portfolio;
- a five-key existing saved view loads on Risk by projection;
- updating it on Risk preserves its hidden Portfolio value;
- a new Risk view stores an empty Portfolio value;
- Stock and P&L still load the full five-key view.

Update tests/s09_plui.py:

- prepare_pl_filter_frame() reads mapped combined_pl;
- Portfolio options remain available after dashboard_frame loses Portfolio;
- unmapped Portfolio values are excluded from P&L filter options;
- P&L Portfolio filtering and the editor source remain unchanged.

Update tests/s29_archive.py:

- the snapshot fixture supplies combined_pl;
- dashboard_frame may be Portfolio-free;
- the written risk.parquet still contains Portfolio;
- multiple Portfolio rows remain distinct in the archive;
- historical P&L projection still returns Portfolio totals.

Update tests/s07_integration.py:

- the committed dashboard is smaller than mapped combined_pl for a repeated
  Portfolio fixture;
- refresh metrics report the reduced dashboard row count;
- SearchCatalog builds successfully from the Portfolio-free dashboard;
- a failed aggregation or search build retains the previous good revision.

Update tests/s10_reads.py and tests/s16_refreshshell.py:

- Risk mount reads compact control metadata plus dashboard_frame;
- Risk mount does not request a full snapshot, pl_snapshot, or combined_pl;
- the lazy New Trades path may still request combined_pl.

Run:

~~~powershell
python -m pytest tests/s14_reporting.py -q
python -m pytest tests/s19_riskfilters.py -q
python -m pytest tests/s23_savedviews.py -q
python -m pytest tests/s09_plui.py -q
python -m pytest tests/s29_archive.py -q
python -m pytest tests/s07_integration.py -q
python -m pytest tests/s10_reads.py -q
python -m pytest tests/s16_refreshshell.py -q
python -m pytest -q
python -m ruff check .
python -m ruff format --check .
git diff --check
~~~

#### 13. Validate with the real Credit-sized data

After a successful cold refresh, inspect both committed grains:

~~~python
import pytest

snapshot = manager.snapshot

mapped = snapshot.combined_pl.loc[
    snapshot.combined_pl["Portfolio Mapped"].eq(True)
]
dashboard = snapshot.dashboard_frame

print("mapped combined_pl rows:", len(mapped))
print("dashboard rows:", len(dashboard))
print("reduction:", round(len(mapped) / len(dashboard), 2))

assert "Portfolio" in mapped
assert "Portfolio" not in dashboard
assert dashboard["Risk"].sum() == pytest.approx(
    mapped["Risk"].sum()
)
assert dashboard["PL"].sum(min_count=1) == pytest.approx(
    mapped["PL"].sum(min_count=1)
)
~~~

Then verify:

1. Cold start reaches revision 1 without reconnecting.
2. The final combining/validation/publish stage is shorter.
3. Risk Explorer opens and expands smoothly.
4. Aggregate P&L totals agree with the detailed mapped frame.
5. Quick Risk works and has no Portfolio dimension.
6. Stock still has its Portfolio filter.
7. P&L still has its Portfolio filter, SOG choices, and Portfolio choices.
8. Unmapped books still show their Portfolio identities.
9. A new official archive still writes Portfolio into risk.parquet.
10. Clearing cache and refreshing produces the same totals as cold start.

The reduction will not always equal the Portfolio count. Rows with different
governed reporting fields are intentionally separate.

#### 14. Use this shortest failure guide

~~~text
dashboard still contains Portfolio
  -> to_dashboard_frame is returning the detailed archive projection

dashboard Risk/P&L is too small
  -> a non-additive value was dropped or additive values were not summed

Open/Current scales with Portfolio count
  -> quotes were summed; retain one validated quote instead

missing P&L becomes zero
  -> use sum(min_count=1), not ordinary groupby sum

commodity rows lose their disabled status
  -> preserve Market Data Status instead of recreating a generic status

cold start fails while building SearchCatalog
  -> remove Portfolio from Quick Risk registries and required columns

saved-view callback says field keys do not match
  -> use the five-key repository plus the four-key Risk projection

Stock or P&L loses Portfolio
  -> keep their global five-field FILTER_DIMENSION_FIELDS contract

P&L Portfolio dropdown is empty
  -> populate options from mapped combined_pl, not dashboard_frame

unmapped callback raises while finding portfolio
  -> remove its former Portfolio filter dependency

official archive says Portfolio is missing
  -> archive the mapped Portfolio projection of combined_pl, not dashboard_frame

Risk page still copies combined_pl during mount
  -> use control_snapshot plus read_frame("dashboard_frame")
~~~

#### 15. Rollback

This fix requires no destructive data migration:

- existing saved-view JSON remains valid under the five-key repository;
- existing historical leaves remain readable;
- new risk.parquet leaves keep the same Portfolio-grain structure;
- raw connectors and combined_pl do not change.

To roll back, revert the code commit. Do not delete saved views or historical
Parquet files. The main behavior to compare before and after rollback is
dashboard row count and Risk interaction time; financial totals must remain
unchanged.
