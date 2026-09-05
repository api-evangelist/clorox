---
name: Look up Clorox product safety and ingredient disclosures
description: >-
  Retrieve Safety Data Sheets, cleaning-product ingredient disclosures and CPSC general
  conformity certificates from The Clorox Company's undocumented first-party REST endpoints.
api: contracts/clorox-tcc-v1-routes.json
base_url: https://www.thecloroxcompany.com/wp-json
operations:
  - GET /tcc/v1/sds-xml
  - GET /tcc/v1/cleaning_labels
  - GET /tcc/v1/general_conformity
---

# Clorox product disclosure lookup

Clorox publishes no API documentation. Every path, parameter and response shape below was
verified live on 2026-09-05 against the provider's own route descriptor
(`contracts/clorox-tcc-v1-routes.json`) and by calling the endpoints. Do not add parameters
that are not listed here — they are not declared and will be ignored.

## Before you start

- **No credential.** All three reads are anonymous. Do not send an Authorization header.
- **No rate limit is published and no rate-limit header is returned.** Pace yourself; a single
  pass over these endpoints is enough, and the SDS response is ~110 KB.
- **Base URL is the corporate site, not `api.clorox.com`.** `api.clorox.com` is a partner
  MuleSoft gateway that returns 403 and will never answer these calls.

## 1. Safety Data Sheets

```
GET https://www.thecloroxcompany.com/wp-json/tcc/v1/sds-xml
```

Takes no parameters. Returns `application/xml`, **not JSON**:

```xml
<safety-data-sheets>
  <sds><name>Clorox Scentiva Disinfecting Wipes ... (CAN002236)</name><url>https://.../....pdf</url></sds>
  <sds><name>Clorox® In-Dryer Clothing Refresher ... (USA002239 ...)</name><url/></sds>
</safety-data-sheets>
```

Parse it as XML. Two things to handle:

- `<url/>` is frequently **empty** — the sheet is indexed but no document is linked. Skip those
  records rather than treating the empty element as a URL.
- The country is encoded in the parenthesised code inside `<name>` (`CAN…`, `USA…`), not in a
  field of its own.

## 2. Cleaning-product ingredient disclosures

```
GET https://www.thecloroxcompany.com/wp-json/tcc/v1/cleaning_labels
GET https://www.thecloroxcompany.com/wp-json/tcc/v1/cleaning_labels?brands=<brand>&search=<text>
```

Optional parameters, all declared in the route descriptor:

| Parameter | Required | Default |
|---|---|---|
| `country` | no | `United States` |
| `brands` | no | — |
| `search` | no | — |

Returns a JSON array of records with `id`, `title.rendered`, `brands`, `country`, `languages`
and `pdf_list`. The actual disclosure text is in the PDFs referenced by `pdf_list`; the API
returns the index, never the ingredients themselves.

There is **no pagination** — no `page` or `per_page` is declared and the full set comes back in
one body. Do not build a paging loop.

## 3. General conformity certificate

```
GET https://www.thecloroxcompany.com/wp-json/tcc/v1/general_conformity?upc=<upc>&lot=<lot>
```

**Both `upc` and `lot` are required.** Omitting either returns:

```json
{"code":"rest_missing_callback_param","message":"Missing parameter(s): upc, lot",
 "data":{"status":400,"params":["upc","lot"]}}
```

`data.params` names exactly what is missing — read it rather than guessing. You need a real UPC
and lot code off physical packaging; there is no way to list certificates.

## Error handling — the one thing that will catch you out

This API uses **two different, contradictory** error conventions:

1. **WordPress core routes** put the failure in the HTTP status (`400`) with
   `{code, message, data:{status, params}}`. Normal.
2. **Hand-written first-party routes** (`/tcc/v1/dsar/countryRegions`,
   `/clorox-security/v1/is-do-not-share`) return **HTTP 200** with
   `{"status":100,"message":"Missing countryCode","data":[]}`. The `100` is not an HTTP status
   code and is documented nowhere.

So: **never decide success from the status line alone.** After a 200, check whether the body is
an object carrying a `status` field. If it is, and `status` is not a success value, treat it as
an error and read `message`.

## What not to do

- Do not POST anything. The three write routes (`/tcc/v1/survey/submit`,
  `/clorox-security/v1/log`, `/clorox-security/v1/saw-terms-notice`) accept no idempotency key
  and have no cancel, delete or undo operation. A retry is a second write you cannot take back.
- Do not treat this as a supported API. There is no SLA, status page, changelog or deprecation
  policy; it backs a marketing website and can change without notice.
