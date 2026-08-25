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
does not. Where an export posts a 301 as a single line with a receiving plant,
it is mirrored into that plant; MB51-style exports that already post both legs
are used as they are.

Two things the numbers depend on, both handled explicitly:

- **Usage averages are clipped to the history you hold.** With six days of
  movements imported, a "30-day average" divides by six days, not thirty. Every
  average states the window it was really measured over.
- **Open orders with no confirmed delivery date** count towards "on order" but
  are kept off the arrival timeline, since dating them would drop the whole
  quantity onto an arbitrary day. The quantity involved is shown alongside.

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
  snapshots, extended backwards by unwinding the movement history for days with
  no snapshot yet, then a dashed projection ahead. Optionally split by plant.
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
numbers padded with leading zeros. Report title rows above the header, and the
totals row at the bottom, are skipped. Stock exports that list one row per reel
are aggregated to material and plant.

Where a material code reads grade-grammage-width (`3300-110-1950`) and the
export carries no real description, it is spelled out as `3300 · 110 g/m² ·
1950 mm`.

## Where the data lives

In the browser's IndexedDB, and — once Dropbox sync is switched on — in a
single file in your Dropbox, so every computer that connects works from the
same numbers. **Uploaded spreadsheets are never stored**; rows are parsed,
written to the database, and the file is released.

### Setting up sync

On the first computer: Settings → Dropbox sync. Create a Dropbox app (Scoped
access, App folder), give it `files.content.read` and `files.content.write`,
add the page's address as a Redirect URI, and paste the app key. On every other
computer, paste the **same** app key and sign in to the same Dropbox account.

Served over http(s) the sign-in returns to the page by itself. Opened straight
from a file it cannot, so Dropbox shows a code to paste back instead.

### How concurrent changes are handled

The database is pulled, merged and pushed rather than overwritten, and uploads
carry the revision the browser last saw — if another machine has written in the
meantime Dropbox refuses the write, and the app merges before retrying instead
of trampling the other copy.

Merging is possible because of how the data is shaped: movements are
append-only with a natural key, so both sides' rows are unioned; stock
snapshots are keyed by plant and date; the open-order list is a dated snapshot
where the later export simply wins. Only settings, column mappings and the
production plan are single-valued, and there the more recently edited copy
wins. A machine that has been offline for a week loses nothing by reconnecting.

Uploads are gzipped. A year of movements at roughly 120 rows a day is about
12 MB of JSON and around 1 MB compressed.

Sync is optional. Without it the app runs exactly as before, confined to one
browser, and Settings still offers JSON backup and restore.

## Dependencies

Chart.js and SheetJS, both loaded from cdnjs at runtime. Dropbox sync, when
enabled, talks to the Dropbox API directly from the page using PKCE, so there
is no app secret in the source and no server in the middle. The app warns you on
startup if either fails to load. Vendor them into the repo if the site's
network blocks the CDN.
