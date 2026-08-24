# Fix 8 — Uncramp the Risk UI and repair the Statics dropdown

This fixes the two problems shown in the screenshots:

1. Dash tabs become large vertical strips in the narrow Jupyter preview.
2. The Statics Write dropdown collapses until only a few letters are visible.

No callback or data logic needs to change.

## 1. Keep the tabs horizontal

Dash changes tabs to full-width mobile tabs below 800 pixels by default. The
Jupyter preview crosses that breakpoint even when the full browser does not.

Open:

```text
rebirth/pages/risk/s16_view.py
```

Add this argument to each of these five `dcc.Tabs(...)` components:

```python
mobile_breakpoint=0,
```

The component IDs are:

```text
risk-workspace-tabs
table-view-tabs
risk-type-tabs
credit-view-tabs
ir-family-tabs
```

For example:

```python
dcc.Tabs(
    id="risk-type-tabs",
    value=risk_options[0]["value"],
    children=[
        dcc.Tab(label=option["label"], value=option["value"])
        for option in risk_options
    ],
    mobile_breakpoint=0,
    className="workspace-tabs risk-type-tabs",
)
```

Then open:

```text
rebirth/pages/static_data/s02_view.py
```

Add the same argument to `dcc.Tabs(id="static-data-mode", ...)`:

```python
dcc.Tabs(
    id="static-data-mode",
    value="read",
    children=[
        dcc.Tab(label="Read", value="read"),
        dcc.Tab(label="Write", value="write"),
    ],
    mobile_breakpoint=0,
    className="workspace-tabs static-data-tabs",
)
```

`0` disables Dash's automatic full-width mobile-tab layout. Your own CSS now
keeps the tabs in one compact row.

## 2. Give the Risk controls their own layout

In `rebirth/pages/risk/s16_view.py`, find the `html.Div` containing Split,
Sort underlying by, and Options.

Change:

```python
className="controls",
```

to:

```python
className="controls risk-explorer-controls",
```

Open:

```text
assets/s08_promotions.css
```

In `.promotion-generation-controls`, change:

```css
flex: 1 1 520px;
```

to:

```css
flex: 1 1 100%;
```

Replace the old Risk Explorer rules:

```css
.risk-explorer-action-field { grid-column: span 2; }
.risk-explorer-actions { flex-wrap: wrap; }
```

with:

```css
.risk-explorer-controls {
  grid-template-columns: repeat(2, minmax(180px, 1fr));
  align-items: start;
}
.risk-explorer-action-field { grid-column: 1 / -1; }
.risk-explorer-actions {
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 10px 12px;
}
```

Finally, replace the existing bottom media rule with:

```css
@media (max-width: 760px) {
  .risk-explorer-controls { grid-template-columns: 1fr; }
  .risk-explorer-action-field { grid-column: 1; }
  .promotion-generation-controls { grid-template-columns: 1fr; }
}
```

The result is simple:

- normal preview: Split and Sort share the first row; Options uses the full
  second row;
- very narrow preview: all three sections use one readable column.

## 3. Stop the Statics dropdown collapsing

Open:

```text
assets/s06_visuals.css
```

Replace:

```css
.static-data-write-actions .static-data-selector {
  flex: 1 1 320px;
  margin: 0;
}
```

with:

```css
.static-data-write-actions > .dash-dropdown-wrapper {
  flex: 1 1 320px;
  width: 100%;
  max-width: 430px;
  min-width: min(300px, 100%);
}
.static-data-write-actions .static-data-selector {
  width: 100%;
  margin: 0;
}
```

Why this works: in Dash 4 the outer `.dash-dropdown-wrapper` is the item in
the flex row. The existing code sized only the inner dropdown button, so the
outer wrapper was allowed to shrink to almost nothing.

## 4. Restart and check

Stop the current app, start it again, then hard-refresh the browser:

```text
Ctrl+C
python app.py
Ctrl+Shift+R
```

Check these four things:

1. Risk type tabs remain in one horizontal row.
2. IR Delta, Basis, and Vega remain in one horizontal row.
3. Split and Sort are aligned, with Options on its own row beneath them.
4. Statics → Write shows the complete dropdown label and still lets the
   Add row, Save, and Cancel buttons wrap on small screens.

## 5. Optional focused tests

```text
python -m pytest tests/s34_riskpivot.py tests/s39_assets.py tests/s42_statics.py -q
python -m pytest tests/s12_startup.py tests/s19_riskfilters.py -q
```

The completed implementation is also available on the `fix-8` branch of the
Rebirth V4.1 repository.
