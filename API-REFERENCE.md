# API Reference

Complete endpoint documentation for the Mineral Intelligence Benchmark API.

**Base URL:** `https://us-central1-fourmonth-73efe.cloudfunctions.net/api`

All payment-gated endpoints use [x402](https://x402.org) on Base mainnet. Settle with any x402-aware client; the unpaid 402 response carries the exact `accepts[]` `PaymentRequirements` to satisfy.

---

## `GET /api/health` — free

Service heartbeat and endpoint inventory.

**Response (200):**

```json
{
  "ok": true,
  "status": "ok",
  "service": "mineral-intel-benchmark",
  "version": "1.0.0",
  "model": { "vision": "gemini-2.5-flash" },
  "endpoints": {
    "GET /api/benchmark/{commodity}": {
      "description": "USGS benchmark data for copper, gold, or silver",
      "supported_commodities": ["copper", "gold", "silver"],
      "price": "$0.10 USDC on Base",
      "payment": "x402"
    },
    "GET /api/benchmark/ultrasound-grooved-tray": {
      "description": "Founder's attested ultrasound-assisted grooved-tray run data with on-chain provenance",
      "price": "$0.10 USDC on Base",
      "payment": "x402",
      "filters": ["feed_type", "frequency_khz", "min_recovery_pct", "attested_only", "limit", "offset"]
    },
    "extract": {
      "POST /api/extract/estimate": { "description": "Free probe — page count, source type, price quote", "payment": "none" },
      "POST /api/extract/run":      { "description": "Paid assay extraction — Gemini 2.5 Flash, quality-graded", "payment": "x402 dynamic (ladder price)" },
      "GET /api/extract/result/:id":{ "description": "Token-gated extraction result",                         "payment": "none (HMAC token required)" }
    }
  },
  "chain": "base",
  "timestamp": "2026-05-15T18:30:00Z"
}
```

---

## `GET /api/benchmark/:commodity` — $0.10 USDC

USGS Mineral Commodity Summaries data.

- **Live commodities** (data file served): `copper`, `gold`, `silver`
- **Attested commodities** (file rolling out): `cobalt`, `graphite`, `lithium`, `manganese`, `nickel`

**Request:**

```bash
curl -i https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper
```

**Unpaid (402):** standard x402 envelope — `error`, `accepts[]` (network=`base`, asset=USDC, amount=`0.10`, recipient), `paymentExpired`.

**Paid (200):**

```json
{
  "ok": true,
  "data": {
    "commodity": "copper",
    "symbol": "Cu",
    "unit": "metric ton",
    "source": "USGS Mineral Commodity Summaries 2026",
    "source_url": "https://pubs.usgs.gov/periodicals/mcs2026/",
    "publication_date": "2026-01",
    "schema_version": "mineral-intel-benchmark-v1",
    "source_cid": "ipfs://bafkrei...",
    "result_cid": "ipfs://Qm...",
    "attestation_uid": "0x5546f1ab9227f9d73d359ee6065c144c153390627019672e5ac6e774330fbd38",
    "price": {
      "spot_usd_per_lb": 4.2,
      "spot_usd_per_tonne": 9260,
      "year_avg_2025_usd_per_lb": 4.05,
      "year_avg_2024_usd_per_lb": 4.24
    },
    "grade_benchmarks": {
      "open_pit_cutoff_pct_cu": { "low": 0.15, "typical": 0.3, "high": 0.5 },
      "underground_cutoff_pct_cu": { "low": 0.8, "typical": 1.5, "high": 2.5 },
      "high_grade_vein_pct_cu": { "low": 2, "typical": 5 }
    },
    "chemistry": {
      "penalty_elements": ["arsenic", "bismuth", "antimony", "fluorine", "mercury", "lead"],
      "common_associations": ["molybdenum", "gold", "silver", "zinc", "lead", "pyrite"],
      "arsenic_penalty_threshold_ppm": 3000,
      "concentrate_grade": { "typical_pct_cu": 28, "range_pct_cu": [20, 35] }
    },
    "production": {
      "world_mine_output_2025_ktonnes": 22000,
      "top_producers_2025": ["Chile", "Peru", "Congo (Kinshasa)", "China", "United States", "Russia"],
      "us_mine_output_2025_ktonnes": 1200
    },
    "recovery_benchmarks": {
      "flotation_pct": { "low": 80, "typical": 87, "high": 93 },
      "heap_leach_solvent_extraction_pct": { "low": 55, "typical": 72, "high": 82 },
      "smelting_refining_pct": { "typical": 99.5 }
    }
  },
  "provenance": {
    "attestation_uid": "0x5546f1ab9227f9d73d359ee6065c144c153390627019672e5ac6e774330fbd38",
    "source_cid": "ipfs://bafkrei...",
    "result_cid": "ipfs://Qm...",
    "verify_url": "https://base.easscan.org/attestation/view/0x5546f1ab9227f9d73d359ee6065c144c153390627019672e5ac6e774330fbd38",
    "ipfs_gateway": "https://gateway.pinata.cloud/ipfs/Qm...",
    "note": "This response has on-chain provenance. Verify the attestation at the verify_url."
  },
  "meta": {
    "schema_version": "mineral-intel-benchmark-v1",
    "generated_at": "2026-05-15T18:30:00Z",
    "payment_network": "base"
  }
}
```

Inner-`data` shape varies by commodity (gold and silver have additional `purity`, `assay_methods`, `refining` blocks). See [SCHEMA.md](./SCHEMA.md).

**Unsupported commodity (404):**

```json
{
  "error": "Commodity not found",
  "supported": ["copper", "gold", "silver"],
  "message": "'tin' is not a supported commodity. Use one of: copper, gold, silver"
}
```

---

## `GET /api/benchmark/ultrasound-grooved-tray` — $0.10 USDC

First-party gravity-separation run logs.

**Query parameters (all optional):**

| Param | Type | Description |
| --- | --- | --- |
| `feed_type` | `placer_slurry` \| `hardrock_float` \| `mixed` \| `other` | Exact-match feed filter |
| `frequency_khz` | number | Ultrasound frequency exact match (0 = elutriation only) |
| `min_recovery_pct` | number | Floor on `recovery_pct_estimated` |
| `attested_only` | `true`/`false` | Only runs with an on-chain attestation |
| `limit` | number (default 20, max 100) | Page size |
| `offset` | number (default 0) | Page offset |

**Paid response (200):**

```json
{
  "ok": true,
  "device": "ultrasound-grooved-tray",
  "device_description": "Custom ultrasound-assisted gravity separation device using a grooved stainless-steel tray at adjustable angle.",
  "stats": {
    "total_runs": 42,
    "attested_runs": 38,
    "avg_recovery_pct": 64.3,
    "best_recovery_pct": 89
  },
  "pagination": { "offset": 0, "limit": 20, "returned": 20, "total": 42 },
  "runs": [
    {
      "run_id": "2026-06-12-001",
      "date_utc": "2026-06-12T15:42:00Z",
      "site_lat": 48.7,
      "site_lon": -120.4,
      "feed_type": "placer_slurry",
      "feed_mass_kg": 12.4,
      "ultrasound_frequency_khz": 0,
      "ultrasound_power_w": 0,
      "tray_geometry": "mini_duke_elutriator_v1",
      "recovery_pct_estimated": 71,
      "recovered_au_g_measured": 1.2,
      "run_cid": "ipfs://Qm...",
      "attestation_uid": "0x...",
      "schema_version": "mineral-intel-ultrasound-run-v1"
    }
  ]
}
```

Coordinates are intentionally rounded to ±0.1° for OPSEC. See [SCHEMA.md](./SCHEMA.md) for the full v1 field list.

---

## Extract API (beta)

Structured-data extraction from public documents. Token-anchored pricing on a 5-tier ladder. Grade-D results auto-refund 80%.

### `POST /api/extract/estimate` — free

**Request:**

```json
{
  "source_url": "https://example.org/report.pdf",
  "source_type": "pdf_digital"
}
```

`source_type` is optional (server auto-detects from MIME). Supported: `pdf_digital`, `pdf_scanned`, `image`, `csv`, `txt`. `xlsx` and `docx` return 415.

**Response (200):**

```json
{
  "ok": true,
  "estimate_id": "ex_2026-05-14-71fa1113",
  "source_type": "pdf_digital",
  "page_count": 12,
  "quality_floor": "B",
  "cost_breakdown": {
    "est_input_tokens": 18420,
    "est_output_tokens": 2200,
    "model_cost_usdc": "0.0241",
    "margin_x": 5,
    "price_usdc": "0.50"
  },
  "ladder_price_usdc": "0.50",
  "expires_at": "2026-05-15T18:45:00Z"
}
```

### `POST /api/extract/run` — x402 dynamic (ladder)

Requires `X-PAYMENT` header matching the ladder price returned by `/estimate`. Unpaid calls return 402 with the canonical x402 envelope.

**Request:**

```json
{ "estimate_id": "ex_2026-05-14-71fa1113" }
```

**Response (200):**

```json
{
  "ok": true,
  "job_id": "ex_2026-05-14-71fa1113",
  "status": "complete",
  "grade": "B",
  "result_url": "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/extract/result/ex_2026-05-14-71fa1113?token=<hmac>",
  "result_ttl_hours": 24
}
```

Ladder tiers: `0.10`, `0.50`, `1.00`, `2.50`, `5.00` USDC. Grade D ⇒ 80% auto-refund.

### `GET /api/extract/result/:id?token=<hmac>` — free (token)

Returns the `assay-extract-v0.1` document.

```json
{
  "ok": true,
  "job_id": "ex_2026-05-14-71fa1113",
  "grade": "B",
  "result": {
    "doc_type": "usgs_circular",
    "lab": null,
    "report_date": "1968-01-01",
    "samples": [{ "sample_id": "F-1", "au_ppm": 12.4, "site_lat": 48.7, "site_lon": -120.4 }],
    "raw_text_excerpt": "..."
  }
}
```

24-hour TTL; coords rounded ±0.1°.

---

## `POST /api/mcp` — free discovery, paid execution

Model Context Protocol over Streamable HTTP. Stateless (one server instance per request). Built with `@modelcontextprotocol/sdk` ^1.12.

### Initialize

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": { "name": "your-client", "version": "1.0.0" }
  }
}
```

### Tools

| Tool | Input | Cost | Notes |
| --- | --- | --- | --- |
| `ask_sales_agent` | `{ question: string ≤500ch }` | free | Groq-backed natural-language Q&A about this API |
| `get_commodity_benchmark` | `{ commodity: copper\|gold\|silver\|lithium\|cobalt\|nickel\|manganese\|graphite }` | $0.10 | Same payload as REST benchmark; missing data files return a soft error |
| `get_ultrasound_run_data` | `{ attested_only?: boolean, limit?: 1–50 }` | $0.10 | Same payload as REST runs endpoint |
| `estimate_extract` | `{ source_url: string, source_type?: enum }` | free | Calls `/api/extract/estimate` loopback |
| `run_extract` | `{ estimate_id: string }` | discovery only | Returns endpoint + ladder price; MCP cannot carry `X-PAYMENT`, so execute the POST directly with an x402 client |
| `get_extract_result` | `{ job_id: string, token: string }` | free | Calls `/api/extract/result/:id` loopback |

Tools that report a paid price return the standard x402 402 envelope from the underlying REST endpoint when called over MCP without payment — MCP itself does not negotiate payment.

---

## Errors

### 402 — Payment Required

Standard x402 envelope. Example:

```json
{
  "error": "Payment required",
  "accepts": [{
    "scheme": "exact",
    "network": "base",
    "maxAmountRequired": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b3566dA4ef5",
    "payTo": "0x750977976Ab85A4Ce5AAbb2e1a9fc80a633f2769",
    "resource": "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper",
    "description": "USGS copper benchmark",
    "maxTimeoutSeconds": 60
  }],
  "x402Version": 1
}
```

`maxAmountRequired` is USDC base units (6 decimals → `100000` = $0.10).

### 404 — Not Found

```json
{
  "error": "Commodity not found",
  "supported": ["copper", "gold", "silver"]
}
```

### 415 — Unsupported Media

Extract API only — `xlsx`, `docx` blocked.

### 500 — Internal Server Error

Retry after a few seconds; if persistent, open a GitHub issue with timestamp + path.

---

## Caching

| Endpoint | Cache-Control |
| --- | --- |
| `/api/health` | `no-store` |
| `/api/benchmark/:commodity` | `public, max-age=3600` (1 h) |
| `/api/benchmark/ultrasound-grooved-tray` | `no-store` (live data) |
| `/api/extract/*` | `no-store` |
| `/api/mcp` | `no-store` |

---

## See also

- [GETTING-STARTED.md](./GETTING-STARTED.md) — first-time walkthrough
- [EXAMPLES.md](./EXAMPLES.md) — cURL / JS / Python
- [SCHEMA.md](./SCHEMA.md) — v1 field definitions
- [x402-GUIDE.md](./x402-GUIDE.md) — payment protocol
- [VERIFICATION.md](./VERIFICATION.md) — verify attestations on Base
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — common errors
