---
name: "create-blanket-po-sagex3"
description: "Create or duplicate a supplier blanket order (POC object) in Sage X3 via the sagex3 MCP, then add its delivery request lines (POD object). Use for any request about a contract, blanket order, open order or purchase delivery schedule."
---

# Create or duplicate a blanket order (POC) and its deliveries (POD) in Sage X3

Three use cases, which can be combined:

| Case | Trigger | Steps |
|---|---|---|
| **A. Creation** | "create a blanket order for supplier X, item Y…" | 1 → 6, then 7 if deliveries |
| **B. Duplication** | "duplicate / copy / recreate blanket order POC…" | 1b → 6, then 7 if deliveries |
| **C. Deliveries only** | "add deliveries / delivery requests on POC…" | 7 only |

This skill is validated by real, successful creations on the SEED folder
(`POCFR0110016`: POC creation + 3 POD deliveries). Unverified points are flagged as
such.

X3 messages are quoted verbatim in French (as returned by the server), with an
English gloss. Reply to the user in their own language.

## Non-negotiable rules

- **Irreversible writes.** Never call `x3_object_create` or `x3_object_modify`
  without explicit user confirmation of: supplier, site, validity dates, currency,
  and for each item line, planned quantity and price; for deliveries, each period
  and quantity. If the user says "try it" on a test folder, that counts as
  confirmation.
- **Never invent** a supplier, item, site, buyer, price list or address code. Ask
  for any missing value.
- **The less you force, the better.** X3 fills in by itself the buyer, payment
  term, tax rule, carrier, delivery mode, analytics, account, supplier item
  reference and net price. Only force what is listed in step 5.
- **Never force `APPFLG`** (signature status) to bypass the signature workflow. See
  step 6.
- **When in doubt, stop.** An existing blanket order on the same supplier / item
  pair, a price gap versus the source, an unknown item: present the facts and wait.
- **Always end with the X3 link** to each blanket order involved (step 9).
- **One item per blanket order**: the web service cannot create several (step 5).

## Transactions: the point that makes everything fail

| Operation | Object | `Transaction_Code` | Note |
|---|---|---|---|
| List | POC | `STD` or `ALL` | |
| Read | POC | `STD` or `ALL` | **Empty = silent failure** (`success:false, status:0`, no message) |
| **Create** | POC | **`ALL` mandatory** | `STD`: passes the business checks then fails with « Le serveur X3 n'est pas autorisé à demander l'ouverture d'une fenêtre [WOPOCSTD / GESPOC] » (X3 may not open a window for the web service). Empty: « Fichier inexistant » (file not found) |
| List | POD | `ALL` | |
| Read | POD | `ALL` | |
| Add deliveries | POD | **`ALL`**, via `x3_object_modify` | |

Always use `ALL` everywhere: it is the only transaction validated end to end.

## Search tool: `x3_record_list` with a filter

```
x3_record_list(
  Sage_X3_Object_Code = "POC",
  Transaction_Code    = "ALL",
  Data__JSON_         = {"list_filters": "[F:POH]BPSNUM = \"FR053\""}
)
```

| Element | Syntax |
|---|---|
| POC field | `[F:POH]<FIELD>` (the header table is POHEADER): `POHNUM`, `BPSNUM`, `POHFCY`, `ORDREF`, `ORDDAT` |
| POD field | `[F:POC]<FIELD>`: `POHNUM`, `POPLIN`, `BPSNUM`, `ITMREF` |
| Text literal | escaped double quotes: `\"FR053\"` |
| AND / OR | `&` / `\|` (both validated) |
| Case-insensitive | `instr(1, tolower([F:BPS]BPSNAM), \"text\") > 0` |

Reading the result:

- `data.filters.FILRAP(0)` must contain the expression you sent. If it is `1=1`,
  the filter was ignored: do not interpret the result.
- The POC list has an implicit filter `FILTSUP(0) = POHTYP=2` (blanket orders).
- `data` may come back as a **JSON string** that must be parsed again. `values` is
  an array of arrays when `list_count > 1`, a flat array when `list_count = 1`.
- POC list: columns `ORDDAT`, `POHNUM`, `POHFCY`, `BPSNUM`, `ORDREF`, sorted by
  descending date, capped at 100 rows.
- POD list: **one row per blanket order item line** (`POHNUM`, `POPLIN`, `BPSNUM`,
  `ITMREF`). This is the only reliable way to enumerate the items of a blanket
  order (see step 1b).

## POC object structure

| Block | Role | Key fields |
|---|---|---|
| `POC0` | Header | `POHFCY` (contract site), `BPSNUM` (supplier), `ORDDAT` (contract date), `ORDREF` (internal ref, 20 chars), `POHNUM` (number, from counter), `REVNUM` (amendment) |
| `POC1` | Terms | `STRDAT` / `ENDDAT` (validity), `CUR`, `BUY`, `PTE`, `VACBPR`, `BPSINV`, `BPRPAY`, `BPAINV` / `BPAPAY`, `INVFCY`, `RCPFCY`, `STOFCY`, `CHGTYP`, `BPTNUM`, `MDL`, `APPFLG` (signed), `CLEFLG` (closed) |
| `POC2` | Item line + price | Line: `ITMREF`, `PRHFCY`, `PUU`, `EXTQTYPUU` (planned qty), `XSTRDAT` / `XENDDAT` (line validity), `FIMHOR` / `FIMHORUOM` (firm horizon), `FRTHOR` / `FRTHORUOM` (planning horizon), `LINPURTYP`. Price: `POPSTRDAT` (price start), `GROPRI` (gross), `DISCRGVAL1..9` (discounts), `NETPRI` (net, computed) |
| `POC3` | Footer | `OCNNUM` / `OCNDAT` / `OCNREM` (supplier order acknowledgment), `INVDTA*` (footer charges / discounts), `TOTVLT` (planned total, computed) |

### Local menus

| Field | Values |
|---|---|
| `APPFLG` (Signed) | 1 No, 2 Partly, 3 Fully, 4 Not managed, 5 Automatic |
| `CLEFLG`, `PRNFLG`, `QUAFLG` | 1 No, 2 Yes |
| `FIMHORUOM`, `FRTHORUOM` | 1 calendar days, 2 working days, 3 weeks, 4 fortnights, 5 months |
| `CHGTYP` | 1 daily rate, 2 monthly rate, 3 average rate, 4 DEB rate, 5 EEC rate |
| `LINPURTYP` | 1 Purchase, 2 Fixed asset, 3 Services |
| `ITMKND` | 1 Standard, 2 Subcontract, 3 Service |

## Step 1 (case A): supplier, site, items

### 1a. Supplier (BPS)

```
x3_record_list(Sage_X3_Object_Code="BPS", Transaction_Code="",
  Data__JSON_={"list_filters": "instr(1, tolower([F:BPS]BPSNAM), \"<lowercase fragment>\") > 0"})
x3_object_read(Sage_X3_Object_Code="BPS", Identifier="<CODE>")
```

Check `BPS0.ENAFLG = 2` (active). Note for control only, without forcing them:
`BPRS.CUR` (currency), `BPS2.PTE`, `BPS2.VACBPR`, `BPS2.BPRPAY`.

### 1b. Site

`POHFCY` is the contract site, given by the user, never inferred from the supplier
code. **The default receiving site and invoicing site may belong to another
company**: they are therefore forced to `POHFCY` (step 5), unless told otherwise.

### 1c. Items

```
x3_supplier_item_code(BPSNUM="<SUPPLIER_CODE>")   # supplier references -> ITMREF
x3_item_search(item="<ITMREF>")                    # existence and unit
```

Item not found: stop (creation via `create-item-sagex3`).

### 1d. Duplicate check (mandatory)

Does an active blanket order already exist on the same supplier / item pair?

```
x3_record_list(Sage_X3_Object_Code="POD", Transaction_Code="ALL",
  Data__JSON_={"list_filters": "[F:POC]BPSNUM = \"<SUPPLIER>\" & [F:POC]ITMREF = \"<ITMREF>\""})
```

For each result, read the order (`x3_object_read POC, ALL`) and check
`POC1.ENDDAT` and `POC1.CLEFLG`. If an open order covers the same period: present
number, period, quantity, price, and ask for instructions. X3 does not block: it
only raises the non-blocking alert « L'article fait l'objet d'une commande ouverte »
(the item already has a blanket order).

## Step 1b (case B): read the source order

```
x3_object_read(Sage_X3_Object_Code="POC", Identifier="<SOURCE_POHNUM>", Transaction_Code="ALL")
```

### Trap: the read does not return the whole item grid

On an order with several items, `POC2` shows **the fields of a single line only**
(scalars), while `POC2.XITMREF` lists all items and `POC2.NBART` their count.
Observed: `POCFR0110001` has `XITMREF = ["DIS001","DIS009"]` but only shows DIS009.

To enumerate lines reliably:

1. `x3_record_list POD` filtered on `[F:POC]POHNUM = \"<SOURCE_POHNUM>\"`: gives
   each `POPLIN` and its `ITMREF`.
2. `x3_object_read POD` on `<POHNUM>~<POPLIN>` (transaction `ALL`) for each line:
   gives `ITMREF`, `EXTQTYPUU`, `PUU`, `PRHFCY`, `ENDDAT`, and existing deliveries.
3. Price (`GROPRI`, discounts) and horizons are only readable for the line shown
   by the POC read. For the other lines: **ask** for the prices, never copy another
   line's price.

### What to copy, what not to copy

| Copy from the source | Do NOT copy |
|---|---|
| `POC0.POHFCY`, `POC0.BPSNUM` | `POHNUM` (counter), `REVNUM` |
| `POC1.CUR` | `APPFLG`, `CLEFLG`, `INVFLG`, `RCPFLG`, `PRNFLG` (statuses) |
| `POC1.RCPFCY`, `POC1.INVFCY` (to force) | `POC1.BUY` (the buyer may no longer exist: observed « Utilisateurs : XSM\Fiche inexistante », user record not found) |
| `POC2.ITMREF`, `PRHFCY`, `PUU`, `EXTQTYPUU` | `NETPRI`, `PRIVLT`, `TOTVLT` (computed) |
| `POC2.FIMHOR`, `FIMHORUOM`, `FRTHOR`, `FRTHORUOM` | `LINACC`, `CCE*`, `ITMREFBPS`, `ITMDES` (auto) |
| `POC2.GROPRI` **and** non-zero `DISCRGVAL1..9` | `POC3.OCN*` (acknowledgment specific to the source) |
| | Dates: always **new**, never the source's |

**Discounts:** they are not reapplied automatically. Source `POCFR0110001` had 10 %
(`GROPRI 143.20`, `NETPRI 128.88`); the copy created without `DISCRGVAL1` came out
at `NETPRI 143.20`. For a faithful duplication, explicitly send every non-zero
`DISCRGVALn`.

**Default dates for a duplication:** unless told otherwise, `ORDDAT` = `STRDAT` =
`POPSTRDAT` = `XSTRDAT` = today, and the same validity length as the source (source
`ENDDAT` minus source `STRDAT`). State them before creating.

`ORDREF`: do not reuse the source's as is (risk of confusion); ask for one or
propose a distinct reference.

**Source with several items:** a faithful duplication into a single order is
impossible (see "Several items", step 5). Propose one blanket order per source item
and wait for agreement.

Finally, note the source's deliveries (step 1b, point 2) and ask whether they should
be copied (with shifted periods) or not.

## Step 2: dates and consistency

Observed rules:

- `XSTRDAT` / `XENDDAT` (line validity) must fall **within** `STRDAT` / `ENDDAT`.
  If `XENDDAT` is omitted, X3 proposes an out-of-period date and blocks with
  « Date incorrecte : date au delà de la période » (date beyond the period) /
  `[M:POC2]XENDDAT`. Always send them.
- `POPSTRDAT` (price start): use the line validity start date.
- Format `YYYY-MM-DD` everywhere.

## Step 3: summary for validation

Present one table: supplier (code + name), site, currency, validity, internal ref,
then per line: item, description, planned quantity, unit, gross price, discount,
expected net price, horizons. For a duplication, add a "source" column and flag
every difference. Wait for confirmation.

## Step 4: signature management check (reminder)

If the user plans to add deliveries, warn right away that the POD will be refused
if the order comes out with `APPFLG = 1` or `2` (see step 6).

## Step 5: build the payload

### Validated minimal payload (1 item)

```json
{
  "POC0": {
    "POHFCY": "<site>",
    "BPSNUM": "<supplier>",
    "ORDDAT": "YYYY-MM-DD",
    "ORDREF": "<internal ref, 20 chars max>"
  },
  "POC1": {
    "STRDAT": "YYYY-MM-DD",
    "ENDDAT": "YYYY-MM-DD",
    "CUR": "<currency>",
    "RCPFCY": "<site>",
    "INVFCY": "<site>"
  },
  "POC2": {
    "ITMREF": "<item>",
    "PRHFCY": "<receiving site>",
    "PUU": "<purchase unit>",
    "EXTQTYPUU": 80,
    "XSTRDAT": "YYYY-MM-DD",
    "XENDDAT": "YYYY-MM-DD",
    "FIMHOR": 16, "FIMHORUOM": 1,
    "FRTHOR": 2,  "FRTHORUOM": 5,
    "POPSTRDAT": "YYYY-MM-DD",
    "GROPRI": 143.2
  }
}
```

Optional fields, only if provided or taken from the source: `POC1.BUY` (check that
the X3 user exists), `POC2.DISCRGVAL1..9`, `POC2.LINPURTYP`, `POC3.OCNNUM` /
`OCNDAT` / `OCNREM`.

### What NOT to send: X3 fills it in

Observed on `POCFR0110016`, created without these fields:

| Field | Value obtained |
|---|---|
| `POC1.BUY` | `AGERA` (web service user) |
| `POC1.PTE` | `SCT30NETEOM` (supplier record) |
| `POC1.VACBPR` | `FRA` |
| `POC1.BPSINV`, `BPRPAY`, `BPAINV`, `BPAPAY` | `FR053`, `FR053`, `001`, `001` |
| `POC1.BPTNUM`, `MDL` | `FR201`, `ES02` |
| `POC1.CCE`, `POC2.CCE1`, `CCE3` | `COMM-001`, `COMM-001`, `142` |
| `POC2.LINACC` | `607000` |
| `POC2.ITMREFBPS`, `ITMDES`, `ITMDES1` | supplier catalog / item record |
| `POC2.PLI` | `T20` |
| `POC2.NETPRI`, `PRIVLT`, `POC3.TOTVLT` | computed |

`x3_object_description` declares `BPSINV`, `BPRPAY`, `BPAINV`, `BPAPAY`, `PTE`,
`VACBPR`, `INVFCY`, `RCPFCY`, `STOFCY`, `CHGTYP` as mandatory: this list is
misleading. Only `RCPFCY` and `INVFCY` had to be forced (company issue), and
`STOFCY` stays empty.

### Several items: ONE BLANKET ORDER PER ITEM

**Observed: the web service creates only one item per blanket order.** Screen
`POC2` is a "list + detail" screen: the `XITMREF` / `XPRHFCY` grid lists the items,
and fields `ITMREF`, `EXTQTYPUU`, `PUU`, `XSTRDAT`, horizons… are the detail of
**a single** item (dimension 1). Only the price grid (`POPSTRDAT`, `GROPRI`,
`DISCRGVAL*`) accepts several entries, and they are **price periods of the same
item**, in ascending date order.

Real attempts and results:

| Attempt | Result |
|---|---|
| Detail fields as arrays (`"ITMREF": ["DIS001","DIS009"]`…) | Refused: « Dimension incorrecte : [M:POC2]ITMREF(1) » (wrong dimension) on every field, and « La date doit être comprise entre les dates des lignes précédente et suivante » (date must be between previous and next lines) on `POPSTRDAT(2)` |
| `XITMREF: ["DIS001","DIS009"]` + DIS009 detail as scalars | Created (`POCFR0110017`) but **one item only**: X3 ignores the list and keeps the detail item |
| Adding a 2nd item via `x3_object_modify` on a signed order (`APPFLG = 3`) | Technical failure: « …pas autorisé à demander l'ouverture d'une fenêtre [WOPOCALL] ». Order unchanged. Unverified hypothesis: modifying a signed contract opens the amendment window |

What to do when the user asks for several items:

1. Say so upfront: through this MCP, **one blanket order = one item**.
2. Propose **one blanket order per item** (same supplier, site, dates, internal ref
   suffixed `-1`, `-2`…), each with its deliveries. Wait for agreement.
3. If the user insists on a single order: stop. Possible routes are manual entry in
   X3, a blanket order import template, or an L4G subprogram published as a web
   service. Do not attempt a workaround.
4. Never send `POC2` detail fields as arrays.

## Step 6: create

```
x3_object_create(
  Sage_X3_Object_Code = "POC",
  Identifier          = "",
  Transaction_Code    = "ALL",
  Data__JSON_         = <payload>
)
```

Leave `Identifier` empty: the number comes from the counter (format `POC` + site +
sequence, e.g. `POCFR0110016`). Get it from `messages`
(`{"info": "Objet créé : <POHNUM>"}`) and `data.POC0.POHNUM`.

**Always read `messages`**: X3 stops at the first failing check and returns
`{"error": "<label>"}` then `{"error": "Erreur zone [M:<screen>]<FIELD>"}`. Fixing
one block may reveal the next one: report each block as it comes.

### Observed errors and fixes

| X3 message | Field | Fix |
|---|---|---|
| « Utilisateurs : XXX\Fiche inexistante » (user not found) | `POC1.BUY` | Remove `BUY` (X3 uses the current user) or ask for a valid buyer |
| « Le site n'est pas sur la même société juridique » (site in another legal company) | `POC1.RCPFCY` | Force `RCPFCY` and `INVFCY` = `POHFCY` |
| « Date incorrecte : date au delà de la période » (date beyond the period) | `POC2.XENDDAT` | Send `XSTRDAT` / `XENDDAT` within the header period |
| « …pas autorisé à demander l'ouverture d'une fenêtre [WOPOCSTD] » | (technical) | Transaction `ALL` instead of `STD` |
| « Fichier inexistant » (MSGBOX) | (technical) | Missing transaction: use `ALL` |
| « Dimension incorrecte : [M:POC2]<FIELD>(1) » | `POC2` detail fields | Never send arrays on the item detail: one item per order |
| « La date doit être comprise entre les dates des lignes précédente et suivante » | `POC2.POPSTRDAT(n)` | Price periods in strictly ascending dates |
| « …pas autorisé à demander l'ouverture d'une fenêtre [WOPOCALL] » on `x3_object_modify` POC | (technical) | Modifying a signed blanket order is not supported by the web service: do not retry |

Non-blocking alert: « L'article fait l'objet d'une commande ouverte » (another
blanket order exists on this item). Mention it in the report.

Never retry identically, never bypass a business check without agreement.

### Signature check (APPFLG)

Read `POC1.APPFLG` in the creation response:

| Value | Consequence |
|---|---|
| 3 Fully / 4 Not managed | POD deliveries allowed |
| 1 No / 2 Partly | **Any POD modification is refused**: « Contrat partiellement signé.\Modification ou suppression interdite » (contract partly signed, change or deletion forbidden) |

Observed: with signature management active, web service creation comes out at
`APPFLG = 1`. After disabling the parameter, web service creation comes out at `3`;
manual entry in X3 comes out at `4`. Both allow deliveries. The parameter change was
not picked up immediately by the web service pool: several successive creations
stayed at `1` before switching to `3`.

If `APPFLG` is 1 or 2 and deliveries are requested:

1. Do not retry the POD in a loop, do not recreate the order in a loop.
2. Do not force `APPFLG` via `x3_object_modify` without explicit agreement.
3. Propose: sign the order in X3 (normal workflow), or have the blanket order
   signature settings checked (folder, company and site levels) and the web service
   pool restarted, then retry.
4. An order already created keeps its `APPFLG`: fixing the settings does not unlock
   it retroactively.

## Step 7: add delivery lines (POD)

### Principle

You do not **create** a POD: X3 automatically creates one empty POD record per
blanket order item line (`NBLIG = 0`). Adding deliveries = **modifying** that
record.

- Identifier: `<POHNUM>~<POPLIN>` (e.g. `POCFR0110016~1`). `POPLIN` comes from the
  POD list filtered on `[F:POC]POHNUM`.
- Transaction: `ALL`.

### 7a. Read the POD

```
x3_object_read(Sage_X3_Object_Code="POD", Identifier="<POHNUM>~<POPLIN>", Transaction_Code="ALL")
```

Note `ITMREF`, `EXTQTYPUU` (contract planned qty), `PUU`, `PRHFCY`, `ENDDAT`,
`NBLIG` and existing arrays. Here, unlike POC, the read does return **all** grid
lines (7 lines observed on `POCFR0110001~2`).

### 7b. Delivery grid

Grid fields (arrays aligned by index, one entry per delivery):

| Field | Role | Values |
|---|---|---|
| `QTYPUU` | Quantity in purchase unit | numeric, mandatory |
| `WRCPDAT` | Requested period (text) | Month: `MM-YYYY` (**validated**, e.g. `11-2026`). Week: `WWW-YYYY` (e.g. `007-2021`, read only, not validated on write). Day: `DD Month` in French (e.g. `30 Janvier`, read only, not validated on write) |
| `WIPSTA` | Status | 1 Firm, 2 Planned, 3 Suggested, 4 Closed |
| `XPRHFCY` | Receiving site | e.g. `FR011` |
| `XFCYADD` | Site address | e.g. `001` (= `POC2.FCYADD`) |

Filled in by X3: `LINSTA` (1 Pending, 2 Late, 3 Closed), `NBLIG`.

Defaults: monthly cadence, status 2 (Planned), site and address of the blanket order
line. A firm delivery (`WIPSTA = 1`) commits the supplier: only use it on explicit
request.

### 7c. Validated payload

```json
{
  "POD1": {
    "QTYPUU":  [20, 20, 20],
    "WRCPDAT": ["11-2026", "12-2026", "01-2027"],
    "WIPSTA":  [2, 2, 2],
    "XPRHFCY": ["FR011", "FR011", "FR011"],
    "XFCYADD": ["001", "001", "001"]
  }
}
```

```
x3_object_modify(
  Sage_X3_Object_Code = "POD",
  Identifier          = "<POHNUM>~<POPLIN>",
  Transaction_Code    = "ALL",
  Data__JSON_         = <payload>
)
```

All arrays must have **the same length**.

### 7d. Checks before sending

- **Sum of quantities** vs `EXTQTYPUU`: if the sum exceeds the contract planned
  quantity, flag it and get confirmation (X3 behavior not verified).
- **Periods** within the blanket order `STRDAT` / `ENDDAT`.
- **POD already filled (`NBLIG > 0`)**: NOT VERIFIED. To be safe, resend the full
  grid, existing lines unchanged (same order) followed by the new ones, then check
  `NBLIG` in the response. Never modify a line with `LINSTA = 3` (closed).
- Existing orders with several items (entered in X3): one POD modification per line
  (`~1`, `~2`…).

## Step 8: verify

```
x3_object_read(POC, "<POHNUM>", "ALL")
x3_object_read(POD, "<POHNUM>~<POPLIN>", "ALL")   # for each line
```

| Block | Check |
|---|---|
| `POC0` | `POHNUM`, `BPSNUM`, `POHFCY`, `ORDREF` |
| `POC1` | `STRDAT`, `ENDDAT`, `CUR`, `RCPFCY`, `INVFCY`, **`APPFLG`**, and auto fields: `BUY`, `PTE`, `VACBPR` |
| `POC2` | `NBART`, `ITMREF`, `EXTQTYPUU`, `GROPRI`, discounts, **`NETPRI`** (compare to the expected net price, especially for a duplication), `XSTRDAT` / `XENDDAT` |
| `POC3` | `TOTVLT` = sum(`EXTQTYPUU` × `NETPRI`) |
| `POD1` | `NBLIG`, `QTYPUU`, `WRCPDAT`, `WIPSTA`, `LINSTA` |

For a duplication, present a source / copy table and flag every difference (net
price, discount, buyer, payment term…).

## Step 9: direct link to the document (MANDATORY)

**Mandatory at the end of the report**, for every blanket order created, duplicated,
viewed or whose deliveries were modified. One link per blanket order involved
(several orders = several links).

### URL construction

| Segment | Value |
|---|---|
| Function | `GESPOC` |
| Transaction number | `1` |
| Object | `OBJPOC` |
| Mode | `M` |
| Identifier | `POC0.POHNUM` returned by X3 |

Template:

```
https://xxxx/syracuse-main/html/main.html?url=/trans/x3/erp/SEED/$sessions?f=GESPOC/1/OBJPOC/M/<POHNUM>
```

### Rendering

Always as a Markdown link, never the raw URL, including in a Teams message:

```
[View POCFR0110016 in X3](https://xxxx/syracuse-main/html/main.html?url=/trans/x3/erp/SEED/$sessions?f=GESPOC/1/OBJPOC/M/POCFR0110016)
```

POD deliveries are viewed from the blanket order: give the order link, not a POD
link.

Only build the link with the `POHNUM` **actually returned by X3**, never an assumed
number. The template is taken from other X3 documents in the folder; if the user
reports that it does not open, say so and fix this skill.

## Final report

Concise, one line per point: blanket order number, supplier, site, validity,
currency, internal ref; per item: planned quantity, gross price, discount, net
price, planned total; notable auto-filled values (buyer, payment term); `APPFLG`
status; X3 alerts; deliveries added (period, quantity, status); for a duplication,
the gaps with the source; **the X3 link to each order (step 9, mandatory)**. Mention
any test order created twice or to be cleaned up.

## Full verified example

Simple duplication of `POCFR0110001` (FR053, DIS009) on the SEED folder:

```json
{
  "POC0": {"POHFCY": "FR011", "BPSNUM": "FR053", "ORDDAT": "2026-09-30", "ORDREF": "TEST-POC-MCP-4"},
  "POC1": {"STRDAT": "2026-09-30", "ENDDAT": "2027-09-30", "CUR": "EUR", "RCPFCY": "FR011", "INVFCY": "FR011"},
  "POC2": {"ITMREF": "DIS009", "PRHFCY": "FR011", "PUU": "UN", "EXTQTYPUU": 80,
           "XSTRDAT": "2026-09-30", "XENDDAT": "2027-09-30",
           "FIMHOR": 16, "FIMHORUOM": 1, "FRTHOR": 2, "FRTHORUOM": 5,
           "POPSTRDAT": "2026-09-30", "GROPRI": 143.2}
}
```

Result: `POCFR0110016`, `APPFLG = 3`, `NETPRI 143.20` (the source's 10 % discount
was not sent, so not carried over), `TOTVLT 11,456`. Then POD `POCFR0110016~1`
modified with 3 planned deliveries of 20 UN (`11-2026`, `12-2026`, `01-2027`):
`NBLIG = 3`, `LINSTA = [1,1,1]`.

Several-items test (DIS001 + DIS009): `POCFR0110017` created with DIS009 only (the
`XITMREF` list was ignored), adding DIS001 by modification refused (`WOPOCALL`
window). Hence the "one item per blanket order" rule.
