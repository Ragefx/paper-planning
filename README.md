# Paper Planning

Stock control and planning for paper at a corrugated site. One self-contained
HTML file — open `index.html` in a browser, or serve it over GitHub Pages.
No build step, no server, no install.

## What it does

Three SAP exports go in every day; the app keeps the history and answers the
questions you would otherwise rebuild in Excel each morning.

| Upload | Contents | How it is stored |
| --- | --- | --- |
| Current stock | one row per material and plant | a dated snapshot; re-importing the same date replaces it |
| Movements | material document lines | appended to history, deduplicated |
| Open orders | purchase order lines | replaces the list; only lines with GR quantity 0 are kept |

Movement types handled: **101** goods receipt, **102** reversal, **251**
consumption, **252** reversal, **301/302** plant-to-plant transfer, **411**
consignment to own stock. Signs are taken from the export when it already
distinguishes receipts from issues, and derived from the movement type when it
does not. A 301 carrying a receiving plant is mirrored into that plant.

Plants are configurable. The defaults assume **SI11** is the running plant and
**SI10** is a warehouse: the 7- and 30-day usage averages use SI11 movements
only, the all-time average uses both.

## Views

- **All Materials** — stock per plant, quantity on order, average daily usage
  and days of cover, sortable, with CSV export.
- **Material Forecast** — projection for one material over 14–90 days, with
  current stock and the 7-day, 30-day and all-time usage averages, plus arrival
  markers carrying order number, quantity and supplier.
- **Warehouse Movement** — a month at a time: actual stock from the daily
  snapshots, a dashed projection ahead, optionally split by plant.
- **Production Planning** — enter next month's forecast consumption; it is
  split across materials by their historical usage mix (editable) and run
  forward to show what runs short and when.

## Importing

Drop `.xlsx`, `.xls` or `.csv` files onto the Import tab, several at once. Each
file is classified from its headers and its columns mapped automatically
against English, Slovene and German SAP header names. Anything the app cannot
place, you map by hand once — the mapping is remembered and reused, and can be
reviewed in Settings.

Before anything is written you get a summary of what the commit will change.
Duplicate files are recognised by content hash, and duplicate movement rows by
date, plant, material, movement type, document and quantity, so overlapping
daily exports are safe to upload.

European number formats (`1.234.567,50`) and the `dd.mm.yyyy` date family are
handled, as are Excel serial dates, SAP trailing-minus quantities and material
numbers padded with leading zeros.

## Where the data lives

In the browser's IndexedDB, on the machine that did the importing. **Uploaded
files are never stored** — rows are parsed, written to the database, and the
file is released.

That means the database is tied to one browser profile. Settings has JSON
backup and restore; use it. If more than one person needs the same numbers,
this needs a shared backend instead.

## Dependencies

Chart.js and SheetJS, both loaded from cdnjs at runtime. The app warns you on
startup if either fails to load. Vendor them into the repo if the site's
network blocks the CDN.
