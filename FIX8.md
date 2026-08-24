# Fix 8 — simplest four-column Risk toolbar

Do only these two changes.

## 1. Move the promotion controls into the fourth box

Open:

```text
rebirth/pages/risk/s16_view.py
```

Inside the **Options** box, leave only the `promotion-toggle` and
`region-toggle` buttons. Cut `build_promotion_generation_controls(...)` from
that box and paste it immediately after the Options box as a sibling:

```python
html.Div(
    [
        html.Span("Promotion controls", className="control-label"),
        build_promotion_generation_controls(
            int(initial_snapshot.revision)
            if initial_snapshot is not None
            else 0
        ),
    ],
    className="control-field",
),
```

The outer container must be:

```python
className="controls risk-explorer-controls",
```

## 2. Use the four-column layout

Open:

```text
assets/s08_promotions.css
```

Replace the existing rules for these four classes with:

```css
.risk-explorer-controls {
  grid-template-columns: 0.7fr 0.8fr 1.9fr 1.9fr;
  align-items: start;
}

.risk-explorer-action-field {
  grid-column: auto;
}

.promotion-generation-controls {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px 10px;
}

.promotion-generation-copy {
  grid-column: 1 / -1;
}
```

This gives exactly:

```text
Split | Sort underlying by | Options | Promotion controls
```
