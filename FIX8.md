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

## 2. Put the Risk controls in one four-column row

The required order is:

```text
Split | Sort underlying by | Options | Promotion controls
```

In `rebirth/pages/risk/s16_view.py`, find the `html.Div` containing the Split,
Sort underlying by, and Options fields.

First, keep only `promotion-toggle` and `region-toggle` inside
`risk-explorer-actions`. Move `build_promotion_generation_controls(...)` out
of that Options field.

Immediately after the Options field, add it back as its own fourth field:

```python
html.Div(
    [
        html.Span(
            "Promotion controls",
            className="control-label",
        ),
        build_promotion_generation_controls(
            int(initial_snapshot.revision)
            if initial_snapshot is not None
            else 0
        ),
    ],
    className="control-field risk-promotion-field",
),
```

Then change the outer container from:

```python
className="controls",
```

to:

```python
className="controls risk-explorer-controls",
```

Open `assets/s08_promotions.css` and use these layout rules:

```css
.promotion-generation-controls {
  display: grid;
  grid-template-columns: minmax(0, 1.25fr) minmax(0, 1fr);
  align-items: start;
  gap: 8px 10px;
  min-width: 0;
}

.risk-explorer-controls {
  grid-template-columns:
    minmax(0, 0.7fr)
    minmax(0, 0.8fr)
    minmax(0, 1.9fr)
    minmax(0, 1.9fr);
  align-items: start;
}

.risk-explorer-action-field,
.risk-promotion-field { grid-column: auto; }

.risk-explorer-actions {
  min-height: 39px;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
}

.promotion-generation-copy {
  grid-column: 1 / -1;
  min-width: 0;
}

.promotion-generation-controls .promotion-action-button {
  width: 100%;
  white-space: normal;
}
```

Replace the old bottom media rule with:

```css
@media (max-width: 600px) {
  .risk-explorer-controls {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .promotion-generation-controls { grid-template-columns: 1fr; }
  .promotion-generation-copy { grid-column: 1; }
}

@media (max-width: 400px) {
  .risk-explorer-controls { grid-template-columns: 1fr; }
}
```

At the normal Jupyter preview width, all four fields remain in one row. Only
genuinely small screens collapse to two columns and then one column.

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
3. Split, Sort, Options, and Promotion controls appear as four columns in that
   order.
4. Statics → Write shows the complete dropdown label and still lets the
   Add row, Save, and Cancel buttons wrap on small screens.

## 5. Optional focused tests

```text
python -m pytest tests/s34_riskpivot.py tests/s39_assets.py tests/s42_statics.py -q
python -m pytest tests/s12_startup.py tests/s19_riskfilters.py -q
```

The completed implementation is also available on the `fix-8` branch of the
Rebirth V4.1 repository.
