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
there stopped. Use one of the SI11-only bases for planning.

Two further things the numbers depend on, both handled explicitly:

- **Usage averages are clipped to the history you hold.** With six days of
  movements imported, a "30-day average" divides by six days, not thirty. Every
  average states the window it was really measured over.
- **Open orders with no confirmed delivery date** count towards "on order" but
  are kept off the arrival timeline, since dating them would drop the whole
  quantity onto an arbitrary day. The quantity involved is shown alongside.

Plants are configurable. The defaults assume **SI11** is the running plant and
**SI10** is a warehouse: the 30-, 60- and 90-day usage averages use SI11 movements
only, the all-time average uses both.

## Views

- **Overview** — the landing page. Both plants drawn side by side, each works
  filling bottom-up with how much of its capacity is in use and coloured by the
  reserved status palette: green inside target, amber past it, red over
  capacity, grey when no stock file has been read (an empty plant is unknown,
  not healthy). Under them the site as a whole, the eight materials closest to
  running out, and a list of what actually needs doing — stale data, more paper
  than the site can hold, deliveries to redirect, loads to move, undated order
  lines — each linking to the tab that deals with it. Every figure on it is the
  same number the detailed tab shows; the page is a summary, never a second
  calculation.

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
  current stock and the 30-, 60- and 90-day and all-time usage averages, plus arrival
  markers carrying order number, quantity and supplier.
- **Warehouse Movement** — a month at a time: actual stock from the daily
  snapshots, extended backwards by unwinding the movement history for days with
  no snapshot yet, then a dashed projection ahead. Optionally split by plant.

  Where open lines carry no delivery date the legend names how many are being
  left out; from the SAP purchase-order list there are none.

  Hovering says different things either side of today. On a future day the
  tooltip lists the deliveries expected — order, quantity, supplier. On a past
  day it reports what was actually posted: goods received and paper consumed,
  one total each, with a transfer line only when stock moved between the plants
  (without it, a day where the line jumps has no explanation on the chart).
  These daily figures sum exactly to the month tiles above the chart, and they
  honour the material and plant currently in view. A past working day with
  nothing posted says so, and a day the calendar closes says the plant was
  closed rather than showing zeros.
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

### What an import actually did
"46,000 skipped" reads as either a working dedupe or a broken one until you can
see the span it produced, so the span is stated outright in four places: the
preview before you commit names **the range the file covers** (which is not the
range it will add); the toast on commit and the movements card name **what the
database now holds**, in rows, calendar days and days actually worked; and the
import log keeps the span each import produced, so the history reads as a
record rather than a pile of counts.

Note that `skipped` counts duplicates *within* the file as well as against what
is stored, so a large skip count on a first import is not by itself proof the
rows were already held — the span is.

## Where the data lives
In **IndexedDB**, in the browser profile that imported it — on disk, not in
memory, so it survives closing the browser and rebooting the machine. Dropbox is
opt-in and off until connected; without it the app is entirely local and sends
nothing anywhere. A different browser, or a different Windows user, is a
different profile and therefore a different, empty database.

A browser treats site storage as disposable by default and may evict it under
disk pressure. The app asks for **persistent storage** on load and reports the
answer in Settings — *durable*, *best effort*, or *not reported* on a browser
without the API. The browser decides on its own terms (Chrome from engagement
heuristics, silently; Firefox by asking), so the result is displayed rather than
relied on, and there is an ask-again button. Clearing site data by hand erases
the database whatever the verdict, which is what the JSON backup is for.

## Open orders: two sources
The spreadsheet export (ME2M at item level) has no delivery date, because the
date lives on the schedule line one level down. The app therefore also reads the
**SAP purchase-order list** — ME2M with scope of list `ALLES`, saved through
*System → List → Save → Local File* as a `.txt`. That is a printed report rather
than a table, so it is recognised by its layout and parsed by shape; it needs no
column mapping. Drop it on the Import page like any other file.

It carries, on every line, what the spreadsheet cannot:

- the **schedule-line delivery date**, present whether or not the supplier has
  confirmed, so nothing has to be estimated from a lead time;
- **Still to be delivered**, the true outstanding quantity, which replaces the
  GR = 0 rule and so finally accounts for part-delivered lines;
- the PO creation date, vendor, and material group.

Reels are not cut to order, so most "part delivered" lines are simply a delivery
that closed a few percent light. A remainder under the **short-delivery
tolerance** (Settings, default 30%) is treated as complete rather than as paper
still to come; above it the line stays open at its outstanding quantity. On a
real export that separates cleanly: 58 residual lines averaging under 20% of
their order, and 3 genuine partials at 33%, 50% and 65%.

### Confirmed or only ordered
Neither export answers both questions: the purchase-order list dates every line
but says nothing about acknowledgement, and the spreadsheet carries the
acknowledgement flag but no usable date. So **import both**. The flag is kept in
its own map keyed by PO and item, separate from the order list, and therefore
survives that list being replaced by the other source.

On the Material Forecast and Warehouse Movement charts each arrival marker is
coloured by that state, using the reserved status palette rather than a series
hue because it is a state, not a category: **filled** = the supplier has
confirmed, **hollow amber** = ordered but not acknowledged, **hollow grey** =
no flag held for that line. A day mixing the two takes the weaker of them —
what matters at a glance is that some of the quantity is not promised — and the
tooltip marks every line with ✓, ○ or · and totals the unconfirmed share.

Hovering a day gives the split in words as well as colour — the day's total,
then a line per state present with its quantity and order count, then the
largest few lines each marked ✓, ○ or ·. The state lines always sum to the
day total. Both order tables under the charts carry a **Status** column saying
the same thing, so the information is never colour alone.

A line's own flag always beats the remembered map, since it came from the same
file as the quantity. Nothing is ever assumed confirmed: a line with no flag
reads *unknown*.

**The confirmed date wins.** The two exports carry two different dates: the
spreadsheet's is the date the supplier has *confirmed*, the purchase-order
list's is the date the schedule line *requests*. A promise beats a request, so
each line is drawn on its confirmed date wherever one is held, and falls back to
the requested date otherwise — which is the whole reason the list is imported
for lines nobody has acknowledged yet. The two can be weeks apart, and drawing
the request when a later promise exists puts paper on site before it arrives.

Resolution is a view, not a rewrite: the stored order list keeps exactly what
its file said, and every consumer reads the merged version. Both tabs and the
Import card state how many lines use which date, and the forecast table marks a
fallback date `req`.

**Import order does not matter.** The spreadsheet cannot date every line and the
list cannot flag any, so letting the spreadsheet replace a current list would
trade real delivery dates for nothing — and silently, because undated lines
vanish from every projection rather than appearing as gaps. A spreadsheet
therefore takes only the flags while a current list is held, and says so. The
test is which export a list came from, not how well dated it happens to be: two
spreadsheets a day apart differ by a percent or two of dated lines and the newer
one must still win. Once the held list goes stale the spreadsheet takes over on
its own, so dropping the list export needs no setting.

Seeing no unconfirmed markers means one of two very different things, so both
tabs say which: none are outstanding, or nothing has been imported that could
tell. The note gives the count either way.

**Import → Open orders held** states the whole picture in one card: which export
the list came from and as of when, how many lines are dated, the confirmed /
not confirmed / unknown split, how many flags are held, and which column they
were read from. When nothing can be told apart it names the reason — no
spreadsheet imported, its confirmation column unmapped, or the two files
describing different purchase orders. An unmapped confirmation column is also
called out in the import preview, before committing rather than after.

## The work calendar
Mon–Fri is only most of the answer, so **Settings → Holidays & shutdowns** holds
the rest. Slovenian work-free days are **computed** rather than listed — Easter
moves, and a hardcoded table would quietly expire — from the twelve fixed dates
plus Easter Sunday, Easter Monday and Whit Sunday, derived with the anonymous
Gregorian computus. Days that are a praznik but still worked (Primož Trubar,
Rudolf Maister) are deliberately absent.

Any of those can be marked **we work this day**, and any date can be added by
hand with a reason — a single closure or a range, which is how a collective
shutdown goes in. A hand-added closure always wins over a cancelled holiday.

One function, `dayOff()`, answers the question, and `consumesOn()` and
`isRunDay()` are the only two things that ask it. So the calendar reaches
everything at once: usage-rate denominators, the forecast walk, the warehouse
and stock-balancing projections, and the month's production-day count. The
calendar syncs with everything else.

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
