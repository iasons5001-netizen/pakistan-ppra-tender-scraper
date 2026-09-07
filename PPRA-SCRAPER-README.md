# Pakistan PPRA Tender Scraper

An Apify Actor that collects **publicly available** procurement notices from Pakistan's federal, provincial and regional public procurement portals and normalizes them into a single dataset contract.

Built with Node.js 20, the Apify SDK v3 and Crawlee 3 (CheerioCrawler for server-rendered pages, PlaywrightCrawler only where JavaScript rendering is genuinely required).

---

## Scope and limits

- Only publicly accessible procurement information is collected. The Actor does **not** log in, does **not** solve CAPTCHAs, and does **not** touch authenticated, access-controlled or non-public areas of any portal.
- Documents are recorded by their public URL only — files are not downloaded or re-hosted.
- Restricted procurement categories (weapons, ammunition, explosives, controlled substances and their precursors) are excluded from automated surfacing. This is a keyword filter on titles, descriptions, categories and sectors — it is a surfacing control, **not** a compliance review, and it will not catch every abbreviation. Human review still applies.

---

## Supported sources

| Key | Portal | Entry points | Engine | Verification status (7 September 2026) |
|---|---|---|---|---|
| `federal` | Federal PPRA — EPMS / EPADS | `epms.ppra.gov.pk/public/tenders/active-tenders` | Cheerio | **Verified live.** Listing columns, `?page=N` paging, detail page fields, documents and corrigendum history all confirmed against real records. |
| `punjab` | Punjab PPRA / Punjab e-Procurement | `eproc.punjab.gov.pk/ActiveTenders.aspx`, `/ArchiveTenders.aspx` | Playwright | **Not verified live.** Host was unreachable from the build environment. ASP.NET WebForms grid with `__doPostBack` paging — selectors are best-effort, see *Repairing an adapter*. |
| `sindh` | Sindh PPRA (SPPRA) / PPMS | `ppms.pprasindh.gov.pk/PPMS/public/portal/notice-inviting-tender`, `e.pprasindh.gov.pk/tenderlst` | Cheerio | **Not verified live.** Host returned 503 through the build environment's egress. PPMS also serves a self-signed certificate, so `ignoreSslErrors` is enabled for this source only. |
| `kp` | Khyber Pakhtunkhwa PPRA (KPPRA) | `www.kppra.gov.pk/kppra/activetenders`, `kppra.kp.gov.pk/page_type/tenders` | Cheerio | **Verified live.** Columns confirmed: `Tender No / Tender Description / Procurement Entity / Date of Advertisement / Closing Date / Tender-EOI / Bidding Documents / Action`. |
| `balochistan` | Balochistan PPRA (BPPRA) / ePPS | `bppra.gob.pk/searchnewTender.php`, `eprocurebalochistan.gob.pk` | Cheerio | **Not verified live.** BPPRA returned HTTP 500 and the ePPS host did not resolve from the build environment. Header-driven parsing is in place; confirm on first production run. |
| `ajk` | AJK PPRA | `www.ajkppra.gov.pk/advertisements.php`, `/index.php` | Cheerio | **Verified live.** Columns confirmed: `Procurement / Procurement Title / Publishing Date / Closing Date / Department / Procuring Agency / Download`. |
| `gb` | Gilgit-Baltistan PPRA | `www.gbppra.gov.pk/procurements/{1..5}` | Cheerio | **Verified live.** Columns, `?page=N` paging and `/procurementdetails/{id}` detail URLs confirmed. |

> **Note on the unverified sources.** Their adapters are complete and follow the same contract, but their CSS/column assumptions have not been checked against a live response. Run each one once with `maxItemsPerSource: 5` and inspect the output before relying on it commercially.

---

## Input

All fields are optional. See `.actor/input_schema.json` for the authoritative definitions.

| Field | Type | Default | Notes |
|---|---|---|---|
| `sources` | array | all seven | Source keys from the table above. |
| `keyword` | string | – | All space-separated terms must appear in title / description / reference. `"quoted phrases"` are matched literally. |
| `reference` | string | – | Substring match on reference number or tender id. |
| `description` | string | – | Substring match on the description only. |
| `departments`, `procuringAgencies`, `provinces`, `cities`, `procurementCategories`, `sectors`, `noticeTypes` | array | `[]` | Case-insensitive substring match; any value matches. |
| `publishedFrom`, `publishedTo`, `deadlineFrom`, `deadlineTo` | date | – | `YYYY-MM-DD`. Records with an unparsable date are **kept**, never silently dropped. |
| `activeOnly` | boolean | `true` | Open tenders only — see below. |
| `includeDetails` | boolean | `true` | Visit each public detail page. |
| `includeDocuments` | boolean | `true` | |
| `includeCorrigenda` | boolean | `true` | |
| `includeEvaluationReports` | boolean | `true` | |
| `includeAwards` | boolean | `false` | |
| `maxItems` | integer | `100` | Total cap across all sources; `0` = unlimited. |
| `maxItemsPerSource` | integer | `0` | Per-source cap; `0` = unlimited. |
| `sortField` | string | `closingDate` | |
| `sortAscending` | boolean | `true` | Nulls always sort last. |

**How `activeOnly` decides.** A tender is treated as closed only when the portal status says so (`closed`, `cancelled`, `withdrawn`, `awarded`, `expired`, `archived`, …) **or** the closing date has passed. A tender with no status and no parsable closing date stays visible — the Actor does not hide an opportunity because a portal formatted a date badly.

---

## Output

Every record carries the same fields, in the same order, from every source:

```
source, sourceName, province, tenderId, referenceNumber, title, description,
noticeType, procurementCategory, sector, department, procuringAgency, office,
city, country, status, publishedDate, closingDate, openingDate, remainingDays,
tenderFee, bidSecurity, bidSecurityPercent, bidValidityDays, procurementMethod,
biddingProcedure, workflowType, documents, corrigenda, evaluationReports,
contractAwards, contact, detailUrl, sourceUrl, scrapedAt
```

Data-quality rules:

- **Nothing is invented.** Missing values are `null`; missing collections are `[]`.
- **Dates** are ISO 8601. A value with a time becomes `2026-10-19T14:30:00+05:00` (portal times are Pakistan Standard Time); a date-only value becomes `2026-10-19`. When a date cannot be parsed, the field is `null` and the original text is preserved in a companion field — `closingDateRaw`, `publishedDateRaw`, `openingDateRaw`.
- **Numeric `a/b/c` dates are read day-first** (the Pakistani convention) unless the second component is greater than 12.
- **Both the source reference and the normalized fields are kept**: `tenderId` holds the portal's own identifier, `referenceNumber` holds the buyer's tender / NIT / inquiry number, `sourceUrl` holds the listing page and `detailUrl` the notice page.
- `remainingDays` is whole days from now to end-of-day on the closing date; negative when past.

Documents appear as `{ name, type, url }` and are split into four buckets by type: `documents` (tender / bidding documents and notices), `corrigenda`, `evaluationReports`, `contractAwards`. Types are `tender_notice`, `bidding_document`, `tender_document`, `corrigendum`, `evaluation_report`, `contract_award`, `prequalification`, `other`.

### Example record (Federal PPRA, live)

```json
{
  "source": "federal",
  "sourceName": "Federal PPRA (EPMS / EPADS)",
  "province": "Federal",
  "tenderId": "TS0000011188E",
  "referenceNumber": "SND-2617/26",
  "title": "Domestic Gas Meters CL:250/ G-6",
  "noticeType": "Tender Notice",
  "procurementCategory": "Goods",
  "sector": "Miscellaneous",
  "procuringAgency": "Sui Northern Gas Pipelines Limited (SNGPL)",
  "office": "SNGPL",
  "city": "Islamabad",
  "status": "Published",
  "publishedDate": "2026-09-07",
  "closingDate": "2026-10-20T14:30:00+05:00",
  "remainingDays": 42,
  "bidSecurity": "3,720,000.00",
  "bidValidityDays": 90,
  "procurementMethod": "Competitive Bidding",
  "biddingProcedure": "Single Stage-Two Envelope",
  "workflowType": "Standard Evaluation Process",
  "documents": [
    { "name": "Download Tender Document", "type": "bidding_document", "url": "https://epms.ppra.gov.pk/pdf?file=..." }
  ],
  "corrigenda": [
    { "name": "Corrigendum #3", "type": "corrigendum", "url": "https://epms.ppra.gov.pk/public/tenders/tender-details/TS0000011188E#corrigendum-3",
      "issuedOn": "2026-09-07T12:48:00+05:00", "revisedClosingDate": "2026-09-10" }
  ],
  "contact": { "name": "Nabeel Ishtiaq", "phone": "+92-429-920-1449", "email": "nabeel.ishtiaq@sngpl.com.pk", "address": "1st Floor, 21 Kashmir Road Lahore" },
  "detailUrl": "https://epms.ppra.gov.pk/public/tenders/tender-details/TS0000011188E",
  "sourceUrl": "https://epms.ppra.gov.pk/public/tenders/active-tenders",
  "scrapedAt": "2026-09-07T12:17:13.786Z"
}
```

A `RUN_SUMMARY` record is written to the default key-value store with per-source counts (collected / saved / filtered / duplicates) and the first 25 errors per source.

---

## De-duplication

Primary key `source + ":" + tenderId`; fallback key `source + ":" + normalizedTitle + ":" + normalizedProcuringAgency + ":" + closingDate` when no identifier is exposed. Both keys are registered for every saved record, so the same tender re-encountered without an id on a different listing still collapses to one row.

---

## Pagination

Each source paginates until any of these is true: `maxItems` is reached, `maxItemsPerSource` is reached, no further rows are returned, or (for portals that sort newest-first) the page is already older than `publishedFrom`.

Two portals need special handling, both contained inside their adapter:

- **Punjab** paginates by ASP.NET postback, so the whole listing is walked inside one browser request via `gotoNextPage()`.
- **Gilgit-Baltistan** lists **oldest-first**, so with `activeOnly: true` the adapter reads page 1 only to discover the last page number, then walks backwards. Without this, a small `maxItems` would fill with 2024 notices.

---

## Error handling

- Each source runs in its own crawler with its own request queue. A source that fails — DNS failure, 500, redesign, TLS problem — is logged, recorded in `RUN_SUMMARY`, and the run continues with the remaining sources. This was exercised in testing: with Sindh returning 503 and Balochistan's ePPS host failing DNS, the other five sources still returned data.
- Permanent client errors (400/401/403/404/405/410/451) are marked no-retry on first failure; server errors keep the normal retry budget (3 retries, exponential backoff).
- Adapter-level `parseList` / `parseDetail` exceptions are caught per page, not per run.

---

## Project structure

```
.actor/
  actor.json            Actor manifest
  input_schema.json     Input definitions
  dataset_schema.json   Dataset fields + table views
  output_schema.json    Run output view
src/
  main.js               Orchestrator: per-source runs, limits, filtering, dedupe, push
  adapters/
    base.js             Adapter contract + shared list/detail building blocks
    index.js            Adapter registry
    federal.js  punjab.js  sindh.js  kp.js  balochistan.js  ajk.js  gb.js
  lib/
    normalize.js        Dataset contract, header alias map, table/label extraction
    dates.js            Date parsing (PKT), ISO conversion, ranges, days remaining
    documents.js        Document discovery, classification, bucketing
    filters.js          Filtering, restricted categories, active detection, sorting
    deduplicate.js      Primary + fallback dedupe keys
tests/
  lib.test.js           18 unit tests over the pure layer (`npm test`)
Dockerfile  package.json  README.md
```

---

## Maintainability: repairing an adapter

The dataset contract is written in exactly one place (`normalizeTender()` in `src/lib/normalize.js`). Adapters emit *raw* objects using canonical field names and never build the final record, so a portal adapter can be rewritten without any consumer noticing.

Three levels of repair, cheapest first:

1. **A column was renamed** — add the new header text to `FIELD_ALIASES` in `src/lib/normalize.js`, or to the adapter's own `HEADER_OVERRIDES`. No code change.
2. **The layout changed** — adjust the adapter's `decorate()` / `parseDetail()`. Cells are located by their own markup first and by column index only as a fallback, so an inserted column does not break parsing.
3. **The portal moved or was rebuilt** — rewrite that one file against the contract in `src/adapters/base.js`. Nothing outside `src/adapters/<key>.js` needs to change.

There is also a safety net: when an adapter's table selector stops matching, `pickTenderTable()` scores every table on the page by how many procurement-like headers it has and parses the best one. A portal redesign usually degrades to partial data rather than zero rows.

**Adding a source:** create `src/adapters/<key>.js`, register it in `src/adapters/index.js`, add the key to the `sources` enum in `.actor/input_schema.json`. Done.

---

## Running locally

```bash
npm install
npx playwright install chromium     # only needed for the Punjab source

# Provide input, then run
mkdir -p storage/key_value_stores/default
cat > storage/key_value_stores/default/INPUT.json <<'JSON'
{ "sources": ["federal"], "activeOnly": true, "includeDetails": true, "maxItems": 20 }
JSON

npm start
npm test
```

On Apify, build from the included `Dockerfile` (`apify/actor-node-playwright-chrome:20`) — the Playwright base image is required for the Punjab source; every other source runs on Cheerio in the same image.

---

## Known gaps

- **KPPRA** opens tender details in a JavaScript modal rather than at a public URL, so `detailUrl` is `null` for that source and the internal record id is kept in `portalRecordId`. The notice and bidding-document links are still captured from the listing row.
- **Gilgit-Baltistan** listings expose an "Opening Date" but no advertisement date, so `publishedDate` stays `null` unless the detail page states one. It is deliberately not populated from the opening date.
- **Federal EPMS** publishes corrigendum history as text, not as files; entries are recorded with an anchor back to the detail-page section that states them.
- Sindh, Balochistan and Punjab adapters await a live verification run (see the source table).
- `maxItems` / `maxItemsPerSource` cap **collection**, and filtering happens after collection. On a portal whose page ordering does not match your filter (Gilgit-Baltistan is the usual case), a tight per-source cap can fill with rows that are then filtered out. If a source returns fewer rows than expected, raise its cap before assuming the adapter is broken.
