# Step 3 — Selective bulk Market Open and Current calls

Use this change when a market service can accept a list of Underlyings in one
request. It lets selected Source Types make one Open request and one Current
request per refresh instead of one request per Underlying. Source Types that are
not selected continue to use the existing per-Underlying calls.

For one selected Source Type with 370 Underlyings, the normal successful path
changes from 740 calls (370 Open + 370 Current) to 2 calls (1 Open + 1 Current).
Each Source Type is still independent, so selecting both `credit/delta` and
`credit/vega` produces four calls in total. Retries can add attempts when an
upstream call fails transiently.

This is deliberately an adapter change, not a new cache or a UI change. Risk
scope, dates, validation, P&L calculation, progress reporting, and atomic
snapshot publishing remain owned by `RiskRefreshManager`.

#### 1. Choose only the Source Types that support bulk

Use Source Type values from `PRODUCT_SPECS` in
`rebirth/domain/s02_products.py`, not display names such as `Credit Delta`.
Start with one product and expand only after it passes the tests. For example:

```python
BULK_MARKET_SOURCE_TYPES = frozenset(
    {
        "credit/delta",
        # "credit/vega",  # Enable only when its service supports the same contract.
    }
)

unknown_bulk_source_types = (
    BULK_MARKET_SOURCE_TYPES - set(PRODUCT_SPECS_BY_SOURCE_TYPE)
)
if unknown_bulk_source_types:
    raise ValueError(
        f"unknown bulk market Source Types: {sorted(unknown_bulk_source_types)}"
    )
```

Keep this selection beside connector composition in
`rebirth/services/s05_sources.py`. Do not put product-specific routing in the
refresh manager.

#### 2. Add an explicit bulk connector contract

In `rebirth/domain/s02_products.py`, add this protocol immediately after
`ProductMarketConnector`:

```python
class ProductBulkMarketConnector(Protocol):
    """One source-bound market connector for an ordered Underlying scope."""

    def __call__(
        self,
        source_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
    ) -> pd.DataFrame: ...
```

Then append two optional fields to `ProductConnectorAdapter`:

```python
@dataclass(frozen=True)
class ProductConnectorAdapter:
    risk: Callable[[pd.Timestamp], pd.DataFrame]
    market_open: ProductMarketConnector
    market_status: ProductMarketConnector
    market_open_bulk: ProductBulkMarketConnector | None = None
    market_status_bulk: ProductBulkMarketConnector | None = None
```

Append the fields in that order. Existing positional constructions such as
`ProductConnectorAdapter(risk, opened, current)` will then continue to work.
Do not change one connector argument so that it sometimes means `str` and
sometimes means `tuple[str, ...]`; separate hooks make the behavior clear.
Also update the adapter's existing docstring: selected bulk hooks receive the
complete ordered tuple once, while the required narrow hooks retain their
one-Underlying contract and act as the fallback.

#### 3. Wrap the two site-owned bulk calls

Add thin wrappers in `rebirth/services/s05_sources.py`, or in the site-owned
connector module that replaces this fixture boundary. Replace the two
`YOUR_SITE_...` names and keyword arguments with the real API calls:

```python
def get_market_open_bulk(
    source_type: str,
    open_date: pd.Timestamp,
    underlyings: tuple[str, ...],
    *,
    market_status: str,
) -> pd.DataFrame:
    return YOUR_SITE_OPEN_FUNCTION(
        source_type=source_type,
        source_date=open_date,
        underlyings=list(underlyings),
        market_status=market_status,
    )


def get_market_status_bulk(
    source_type: str,
    market_date: pd.Timestamp,
    underlyings: tuple[str, ...],
    *,
    market_status: str,
) -> pd.DataFrame:
    return YOUR_SITE_CURRENT_FUNCTION(
        source_type=source_type,
        source_date=market_date,
        underlyings=list(underlyings),
        market_status=market_status,
    )
```

The manager supplies the authoritative arguments:

- `open_date` is the checker date, one pandas business day before Market Date.
- `market_date` is the resolved Market Date.
- `underlyings` is ordered, unique, and derived from validated Risk plus any
  approved supplemental market scope.
- `market_status` is exactly `Live` or `OFFICIAL`.

Pass these values through. The connector must not recalculate the date or
inspect the wall clock to choose Live/OFFICIAL. It must return only rows for the
requested tuple. If the only available upstream endpoint deliberately returns
the complete product book, filter it once in this site wrapper by exact Source
Type and requested Underlying; do not deduplicate or aggregate its rows. A
scoped endpoint returning unexpected rows should not be filtered—the manager
must reject that contract breach.

#### 4. Return the same rows as the existing narrow calls

Bulk changes call count only. Concatenate-equivalent data must still be
returned at the existing quote grain:

| Product shape | Open columns | Current columns |
|---|---|---|
| No tenor, for example `fx/delta` | `Underlying, Open` | `Underlying, Current` |
| One tenor, for example `credit/delta` | `Underlying, Tenor Swap, Tenor Swap Order, Open` | `Underlying, Tenor Swap, Tenor Swap Order, Current` |
| Two tenors, for example `ir/deltavega` | `Underlying, Tenor Swap, Tenor Option, Tenor Swap Order, Tenor Option Order, Open` | `Underlying, Tenor Swap, Tenor Option, Tenor Swap Order, Tenor Option Order, Current` |

`Risk Type` and `Risk Greek` may also be returned, but every supplied value must
match the selected `ProductSpec`; the framework inserts the product identity
when those two columns are absent. Current may return `Market Status`, but every
row must exactly match the supplied status.

The existing validators continue to enforce the important rules:

- identity and tenor keys cannot be null or blank;
- `Open` and `Current` must be finite numeric values;
- tenor order values must be non-negative integers and consistent per
  Underlying;
- rows must be unique on `Risk Type, Risk Greek, Underlying` plus the applicable
  tenor columns;
- returning an Underlying that was not requested fails the refresh;
- omitting an unavailable requested Underlying is allowed and is surfaced as
  missing market data rather than zero;
- an empty DataFrame is allowed only when it has the correct columns.

Do not aggregate duplicate quotes, fill missing quotes with zero, or silently
discard unexpected Underlyings in the connector. Those cases must reach the
canonical validation boundary and fail visibly.

#### 5. Register bulk hooks only for the selected adapters

In `_get_csv_product_connector_adapters()`—or the equivalent production adapter
builder—keep the existing `risk`, `market_open`, and `market_status` functions.
Add the optional closures only for a selected Source Type:

```python
bulk_open = None
bulk_current = None

if source_type in BULK_MARKET_SOURCE_TYPES:

    def bulk_open(
        open_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
        _source: str = source_type,
    ) -> pd.DataFrame:
        return get_market_open_bulk(
            _source,
            open_date,
            underlyings,
            market_status=market_status,
        )

    def bulk_current(
        market_date: pd.Timestamp,
        underlyings: tuple[str, ...],
        *,
        market_status: str,
        _source: str = source_type,
    ) -> pd.DataFrame:
        return get_market_status_bulk(
            _source,
            market_date,
            underlyings,
            market_status=market_status,
        )

adapters[source_type] = ProductConnectorAdapter(
    risk=risk,
    market_open=market_open,
    market_status=market_status_connector,
    market_open_bulk=bulk_open,
    market_status_bulk=bulk_current,
)
```

The default `None` is the feature switch. There is no second routing table in
the manager: selected adapters have the hooks, and all other adapters
automatically retain their current per-Underlying behavior.

#### 6. Validate the optional hooks at startup

In `RiskRefreshManager.__init__` in `rebirth/services/s06_refresh.py`, keep the
existing required-hook check, then add:

```python
optional_hooks = ("market_open_bulk", "market_status_bulk")
invalid_optional_hooks = [
    hook
    for hook in optional_hooks
    if getattr(adapter, hook) is not None
    and not callable(getattr(adapter, hook))
]
if invalid_optional_hooks:
    raise TypeError(
        f"connector adapter for {source_type!r} has non-callable optional "
        f"hooks: {invalid_optional_hooks}"
    )
```

This catches incorrect configuration before a cold-start refresh begins.

#### 7. Add one bulk loader with the existing retry policy

Add the following method immediately after `_load_market_frames` in
`RiskRefreshManager`. It makes one logical call for the full tuple, retries only
transient exceptions, and reports real progress without pretending that each
Underlying is a separate call:

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
    failure_message = (
        "Opening bulk market connector failed."
        if stage == "market_open"
        else "Current bulk market connector failed."
    )

    for retry in range(_MARKET_RETRIES + 1):
        retry_message = (
            "" if retry == 0 else f" Retry {retry} of {_MARKET_RETRIES}."
        )
        self._progress_activity(
            connector_name,
            stage,
            source_type=spec.source_type,
            product_index=1,
            product_total=1,
            message=(
                f"Loading {label} for {len(underlyings):,} Underlyings "
                f"in one bulk call.{retry_message}"
            ),
        )
        try:
            frame = connector(
                source_date,
                underlyings,
                market_status=market_status,
            )
        except (TypeError, ValueError):
            self._progress_activity(
                connector_name,
                stage,
                source_type=spec.source_type,
                message=failure_message,
            )
            raise
        except Exception as exc:
            if retry == _MARKET_RETRIES:
                self._progress_activity(
                    connector_name,
                    stage,
                    source_type=spec.source_type,
                    message=failure_message,
                )
                raise
            LOGGER.warning(
                "Bulk market connector failed for %s; retry %d of %d: %s",
                spec.source_type,
                retry + 1,
                _MARKET_RETRIES,
                exc,
            )
            self._sleep(_MARKET_RETRY_DELAY_SECONDS)
            continue

        if not isinstance(frame, pd.DataFrame):
            kind = "market Open" if stage == "market_open" else "current market"
            raise TypeError(f"bulk {kind} connector must return a pandas DataFrame")
        return frame

    raise AssertionError("unreachable bulk market retry state")
```

`TypeError` and `ValueError` remain fail-fast because they normally mean a bad
contract or bad data. Other exceptions retain the current four-retry policy.
Do not add a permanent module-global DataFrame cache around this call; the
refresh manager already owns the last-good immutable snapshot.

#### 8. Route Open and Current through bulk when present

At the start of `_load_product_market_open`, resolve the status and add this
branch before the existing per-Underlying code:

```python
adapter = self._connector_adapters.get(spec.source_type)
selected_status = _require_market_status(market_status)

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

Leave the existing `connector`, `load_one`, and `_load_market_frames` block
immediately below it. That block is the fallback.

Add the matching branch at the start of `_load_product_market_status`:

```python
adapter = self._connector_adapters.get(spec.source_type)
selected_status = _require_market_status(market_status)

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

After adding each branch, remove only the duplicate local assignments of
`adapter` and `selected_status` from the unchanged fallback below it.

The returned bulk frames must still pass through
`get_product_market_open()`, `get_product_market_status()`, and
`_reject_unrequested_market_underlyings()`. Do not bypass or reproduce those
validators in the new route.

#### 9. Make connector-call logging truthful

In the refresh loop, change the Open counter from an unconditional Underlying
count to:

```python
adapter = self._connector_adapters.get(source_type)
market_open_calls += (
    1
    if adapter is not None and adapter.market_open_bulk is not None
    else len(requested_underlyings)
)
```

Make the corresponding Current change:

```python
adapter = self._connector_adapters.get(source_type)
market_status_calls += (
    1
    if adapter is not None and adapter.market_status_bulk is not None
    else len(requested_underlyings)
)
```

These are logical call counts. Retry attempts are already visible through the
warning log and progress message.

#### 10. Add focused tests before using real data

Add the following cases to `tests/s07_integration.py`:

1. A selected adapter receives the complete ordered tuple once for Open and
   once for Current.
2. An adapter with both optional hooks left as `None` still calls its legacy
   Open and Current functions once per Underlying.
3. A transient bulk exception is attempted five times in total: the original
   call plus four retries.
4. A bulk response containing an unrequested Underlying is rejected.
5. Duplicate canonical quote keys are rejected by the existing product market
   validator.
6. An empty, correctly shaped bulk response is accepted as unavailable data.
7. The full refresh publishes no partial revision if either bulk leg fails.

The core routing assertion should look like this:

```python
calls: list[tuple[pd.Timestamp, tuple[str, ...], str]] = []


def bulk_current(
    market_date: pd.Timestamp,
    underlyings: tuple[str, ...],
    *,
    market_status: str,
) -> pd.DataFrame:
    calls.append((market_date, underlyings, market_status))
    return pd.DataFrame(
        {
            "Underlying": list(underlyings),
            "Tenor Swap": ["5Y"] * len(underlyings),
            "Tenor Swap Order": [0] * len(underlyings),
            "Current": [101.0] * len(underlyings),
        }
    )


adapter = ProductConnectorAdapter(
    risk=risk,
    market_open=legacy_open,
    market_status=legacy_current,
    market_status_bulk=bulk_current,
)

# Build a manager with this adapter for "credit/delta", then call or refresh it.
assert calls == [
    (
        pd.Timestamp("2026-08-21"),
        ("CREDIT_A", "CREDIT_B", "CREDIT_C"),
        "OFFICIAL",
    )
]
```

Run the narrow test first, then the full gate:

```powershell
python -m pytest tests/s07_integration.py -q
python -m pytest -q
```

#### 11. Roll out one Source Type and verify the result

1. Enable one Source Type in `BULK_MARKET_SOURCE_TYPES`.
2. Start the app and perform one cold-start refresh.
3. Confirm the progress message says one bulk call with the expected Underlying
   count.
4. Check the selected connector's own call counter or log for one Open and one
   Current call. The manager's refresh metric is aggregate, so its total should
   fall by `2 * (number_of_underlyings - 1)` when one Source Type changes from
   narrow Open and Current calls to bulk.
5. Compare canonically sorted Open and Current keys, tenor orders, and values
   with the previous narrow-call output—not only the row counts—then compare the
   resulting market and P&L values.
6. Check Quick Market, Risk Explorer, and Aggregate P&L for the selected product.
7. Only then add another Source Type.

If anything fails, remove that Source Type from
`BULK_MARKET_SOURCE_TYPES` or omit its two optional adapter hooks. The old
per-Underlying path resumes without changing calculations, pages, saved views,
or historical Parquet files.
