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

### Stock policy

Four levels — minimum, ideal, maximum, over-max — set in **days of production
at the planned rate**, so each material gets its own quantities without anyone
maintaining a table of kilos. Minimum triggers an order and ideal sizes it;
maximum and over-max only warn. Lead time and order multiple sit alongside
them, and all of it lives on the Production Planning tab.

### Working days

The machines run Monday to Friday, so a usage rate is **per working day**, not
per calendar day, and no projection draws stock at a weekend. Consumption
posted on a Saturday still counts towards the total — it is only the divisor
that changes. Cover is still reported in calendar days, since that is what
"when do I run out" means: 12.9 working days of stock on a Tuesday runs out a
fortnight later, not next week.

Settings has a toggle for sites that run seven days.

### Special stock

A movement carrying a special-stock indicator (`K`, vendor consignment) moves
stock the stock report does not count. A 411 posts both legs against one
document — consignment out, own stock in — so counting both nets to zero and
loses the own-stock gain the snapshot actually shows. Special-stock legs are
excluded from stock balances and from the reconstructed history; consumption is
counted whatever the stock type, because it was consumed.

### The all-time usage basis

When the warehouse plant stopped consuming more than a fortnight before the
latest movement, the all-time average is describing a two-site operation that
no longer exists. Selecting that basis says so, and names the date production
there stopped. Use the 7- or 30-day basis for planning.

Two further things the numbers depend on, both handled explicitly:

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

- **All Materials** — stock per plant, quantity on order, average daily usage,
  and two different cover figures, sortable, with CSV export.

  **Days to stockout** is the real answer: stock is run forward day by day at
  the average rate and each open order is added on the day it is actually due,
  so a delivery arriving before the shortfall pushes the date out and one
  arriving after it does not help. It looks 90 days ahead. **Cover without
  deliveries** is plain stock ÷ usage — the exposure if every order slipped —
  and a ↑ marks the materials that only get through because something lands in
  time.

  Two markers sit next to the on-order quantity: `*` for quantity with no
  confirmed delivery date, which cannot be placed on the timeline, and a red
  `!` for orders already past their delivery date, which the projection assumes
  arrive today.
- **Material Forecast** — projection for one material over 14–90 days, with
  current stock and the 7-day, 30-day and all-time usage averages, plus arrival
  markers carrying order number, quantity and supplier.
- **Warehouse Movement** — a month at a time: actual stock from the daily
  snapshots, extended backwards by unwinding the movement history for days with
  no snapshot yet, then a dashed projection ahead. Optionally split by plant.
- **Production Planning** — enter the month's board area and grammage; the app
  works out the paper, splits it across materials by their historical usage mix
  (editable), and returns an order list: what to order, how much, and by when.

  The tonnage is `m² × g/m² ÷ 1000`, plus an optional waste allowance — board
  grammage being the weight of a square metre of finished board, all plies
  together. It is spread over Monday–Friday, over every calendar day, or over
  an **agreed production-day count** — shift calendars are settled ahead of the
  month and need not match either. Where more days are agreed than the month
  has weekdays, weekends are added one per week rather than bunched at one end;
  where fewer are agreed, weekdays are dropped the same way. Whichever is
  chosen, the month consumes exactly the planned tonnage — only the daily rate,
  and so the timing of the orders, changes. Each material is then walked from today to the end of the plan
  month: drawn at its current average until the month starts and at the planned
  rate inside it, with existing open orders arriving on their dates. Whenever
  the balance is about to fall below the **minimum**, an order is proposed
  arriving that day, sized to bring stock back to **ideal** and rounded up to
  the order multiple. Its order-by date is that arrival less the supplier lead
  time; anything already inside the lead time is flagged to go out today.

  Quantities already on order are shown in blue and never ordered again. The
  **maximum** and **over-max** levels never cause an order — they warn, which
  is what matters when a plant is already full and shipping stock back to the
  warehouse.

- **Stock Balancing** — what to move, where to, and when, so neither plant runs
  out of room. Capacity here is **tonnes**, not days of cover: a warehouse fills
  up by weight whatever the line is consuming. Both plants are walked forward
  day by day **per material**, because a transfer instruction has to name what
  goes on the lorry; aggregate tonnes can say a plant is too full but not which
  reels to shift, nor whether the warehouse even holds the grade the line is
  about to run out of.

  Paper moves **both ways**, and the walk models that whether or not anything is
  being proposed:

  - **Back to the production plant**, because paper stored at the warehouse has
    to return to be run. These are not suggestions — it is what happens today,
    and leaving them out is what once made the warehouse fill up for ever and
    the production plant look shorter than it is. A full lorry comes when there
    is room for one; when the plant is already full, only what the line needs
    that day.
  - **Out to the warehouse**, when the production plant is over capacity and
    there is no delivery left to redirect. The grade with the deepest cover goes
    first, and nothing leaves unless the minimum days of it stay on site — so
    nothing is sent away only to be fetched back.

  Redirecting a delivery that has not shipped is always tried first: it costs
  one journey where moving the same paper after it lands costs two, and the
  latest-arriving deliveries go first because those are the ones a supplier can
  still be told about. Everything travelling the same way on the same day rides
  together, so a day of shifting reads as one instruction with a load list.

  Nothing is ever moved that would push the receiving plant past its own
  capacity. When moving cannot solve it, the tab says so with the number that
  matters: the two plants together against their combined capacity, and how many
  tonnes have to be pushed back. Moving paper between plants cannot change how
  much of it there is.

- **History** — every month's consumption read from its 251/252 postings,
  whether or not a plan was saved for it. Save a plan on the planning tab and
  the month is scored against what was actually used; enter the board actually
  produced and the real paper weight per square metre falls out, which is how
  the waste allowance should be set rather than guessed.

## Importing

Drop `.xlsx`, `.xls` or `.csv` files onto the Import tab, several at once. Each
file is classified from its headers and its columns mapped automatically
against English, Slovene and German SAP header names. Anything the app cannot
place, you map by hand once — the mapping is remembered and reused, and can be
reviewed in Settings.

**Any import can be undone**, one at a time, newest first — the log records what
each commit added, replaced or overwrote. Reversing them out of order could
restore a stale stock snapshot over a fresher one, so the log behaves as a
stack; undo history is kept for the last ten imports.

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

## Finding out where a number came from

Every usage figure on the All Materials tab links to the postings behind it —
the window, which days counted as production days, the documents and the
arithmetic. The header shows how old the data is, and any tab that decides
something warns when the newest movement is more than two days back.

The order plan exports to CSV and prints, since it is a list somebody acts on
away from the screen. On a phone the materials table keeps the columns that
answer "what is about to run out" and drops the rest.

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

## Layout

Navigation is a left sidebar rather than tabs across the top, which gives every
page — and every chart — the full width of the window. Below 900px it folds
into a scrolling top bar and the materials table drops to the four columns that
answer "what is about to run out".

The KPI row is a single joined panel rather than separate floating boxes, so it
reads as one instrument cluster above the content. Severity appears as a rail
down the left edge of a table row, before any number has been parsed.

Light is the primary design; dark is an alternate, re-stepped against its own
surface rather than an automatic inversion, and still toggled from the sidebar.

## Colour

Series colours come from a validated categorical palette and are assigned by
**entity, never by rank**: stock is slot 1, the warehouse plant slot 2,
deliveries slot 3. A projection is the same quantity as the actual it continues,
so it shares that colour and is distinguished by a dashed line rather than a
second hue. Both modes were re-validated against this app's own surfaces —
lightness band, chroma floor, colour-vision separation, and contrast all pass on
the all-pairs test.

Status colours (good / warning / serious / critical) are reserved and never used
for a series. They always ship beside a text label, never carrying meaning by
hue alone, and figures set in type use darker text-safe inks since the mark
colours are too light to read as text on white.

## Dependencies

Chart.js and SheetJS, both loaded from cdnjs at runtime. Dropbox sync, when
enabled, talks to the Dropbox API directly from the page using PKCE, so there
is no app secret in the source and no server in the middle. The app warns you on
startup if either fails to load. Vendor them into the repo if the site's
network blocks the CDN.
