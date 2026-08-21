# Report specimens

`report-specimens.html` is the **design reference** for both renderings specified in
[`../report-composition.md`](../report-composition.md). Open it in a browser.

**Every number in it is invented.** It exists so the layout, the translation layer, and
the provenance treatment could be reviewed before the harness produces anything real.
The simulated dataset is internally consistent — the cost figures, the confusion-matrix
counts, and the throughput numbers all derive from one hypothetical 300-invoice run — so
it reads correctly, which is exactly why it carries a persistent warning banner and must
never reach a client.

## Naming

Exhibits are named for **the question they answer**, never by letter or by an internal
label. "Payback timeline" and "Assumption sensitivity" tell a reader what they are looking
at; "Cut A" and "Cut B" require them to hold a key in their head, and analyst shorthand
like *cut* or *slice* does not survive contact with an executive audience. Any exhibit
added later inherits this rule.

## What to reuse when building `analysis/report_executive.py`

- Block order and the two-screen budget
- The four-tile translation (rate → count per 1,000, throughput → machine-hours)
- Provenance badge treatment: measured filled, projected dashed-outline-no-fill
- The methodology strip, including the "not triggered" and "not tested" rows
- Exhibit names that state the question, plus the comparison table pattern used for the
  candidate exhibits — when several charts sit together, say what distinguishes them rather
  than leaving the reader to infer it
- Chart palette: validated categorical slots 1–3 plus the status ramp for state
  segments, both stepped for light and dark surfaces

## Charting constraints this specimen already satisfies

- No dual-axis charts anywhere — the capability panel is small multiples with
  independent scales, sharing one x-axis
- Status colors carry an icon and a label, never hue alone
- Every mark has a `<title>` for hover; wide charts scroll inside their own container
- All colors are tokens defined for light and dark; nothing is defined only inside a
  media query
