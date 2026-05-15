# Data Schema

Specification of the on-chain attestation schemas (v1, locked) and the JSON shapes returned by the REST API.

---

## Overview

| Schema | Version | UID (Base mainnet) | Purpose |
| --- | --- | --- | --- |
| `mineral-intel-benchmark-v1` | 1 | `0xde9e029944dd16f3dad09458ec50b1823e4fbeaa35e6779509aeb84059bb8531` | USGS critical-mineral benchmarks |
| `mineral-intel-ultrasound-run-v1` | 1 | `0x7fe15e2b15e89e5e16348683b62c90c9139fdbf1b24d424d7931b4572334d73d` | Founder gravity-separation run logs |

Both schemas are **locked**. New fields ship as v2 with a new schema UID; v1 stays valid.

Browse on Base easscan: `https://base.easscan.org/schema/view/<schema-uid>`.

---

## Benchmark schema (v1)

### On-chain encoded fields

The EAS attestation encodes the canonical reference set:

| Field | Type | Notes |
| --- | --- | --- |
| `commodity` | string | e.g. `copper` |
| `schema_version` | string | `mineral-intel-benchmark-v1` |
| `source_cid` | string | IPFS CID of USGS source PDF |
| `result_cid` | string | IPFS CID of normalised JSON |
| `publication_date` | string | `YYYY-MM` |

### REST data payload

The REST endpoint returns the full normalised JSON (richer than the encoded attestation). The shape is commodity-specific but every payload has these top-level keys:

| Key | Description |
| --- | --- |
| `commodity` | Mineral name (lowercase) |
| `symbol` | Element symbol (`Cu`, `Au`, `Ag`, …) |
| `unit` | Reporting unit (`metric ton`, `troy ounce`, …) |
| `source` | Citation string |
| `source_url` | USGS MCS publication URL |
| `publication_date` | `YYYY-MM` |
| `schema_version` | `mineral-intel-benchmark-v1` |
| `source_cid` | `ipfs://...` (PDF) |
| `result_cid` | `ipfs://...` (JSON) |
| `attestation_uid` | Per-record attestation UID on Base |
| `price` | Object — spot + recent annual averages |
| `grade_benchmarks` | Object — cutoff grades per deposit type |
| `chemistry` | Object — penalty elements, associations, concentrate grades |
| `production` | Object — world + US mine output (latest year) |
| `recovery_benchmarks` | Object — process-specific recovery ranges |

Per-commodity additions (non-exhaustive):

- Gold / silver: `purity`, `assay_methods`, `refining`
- Lithium: `brine_vs_hardrock`, `co2e_factor_kg_per_kg_lce`
- Cobalt / nickel: `byproduct_credits`, `payable_metal_pct`

See live payloads via `GET /api/benchmark/{copper|gold|silver}` (paid).

### Supported commodities

Attested on-chain (all 8): copper, gold, silver, cobalt, graphite, lithium, manganese, nickel.

Live data file served via REST: copper, gold, silver. The remaining 5 are queued for deployment refresh.

### Example record (copper, trimmed)

```json
{
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
  "price": { "spot_usd_per_lb": 4.2, "spot_usd_per_tonne": 9260 },
  "grade_benchmarks": {
    "open_pit_cutoff_pct_cu": { "low": 0.15, "typical": 0.3, "high": 0.5 }
  },
  "production": {
    "world_mine_output_2025_ktonnes": 22000,
    "us_mine_output_2025_ktonnes": 1200
  }
}
```

---

## Run-log schema (v1)

Single ultrasound-assisted gravity-separation run on a grooved stainless tray (Phase 3) or mini-Duke elutriator (Phase 2 baseline).

### Fields

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `run_id` | string | ✓ | Format `YYYY-MM-DD-NNN` (zero-padded) |
| `date_utc` | string | ✓ | ISO 8601 UTC start time |
| `site_lat` | number |   | Rounded ±0.1° for OPSEC |
| `site_lon` | number |   | Rounded ±0.1° for OPSEC |
| `feed_type` | enum | ✓ | `placer_slurry` \| `hardrock_float` \| `mixed` \| `other` |
| `feed_mass_kg` | number |   | Total feed mass |
| `feed_grade_ozt_per_ton_estimated` | number |   | Troy oz / short ton |
| `slurry_water_ratio` | string |   | e.g. `"1:3"` |
| `ultrasound_frequency_khz` | number | ✓ | 0 = elutriation only |
| `ultrasound_power_w` | number |   | Watts (0 if off) |
| `duty_cycle_pct` | number |   | 0–100 |
| `tray_geometry` | enum |   | `mini_duke_elutriator_v1` \| `grooved_stainless_v1` \| `other` |
| `tray_angle_deg` | number |   | 0–90 |
| `run_duration_min` | number |   | Minutes |
| `concentrate_mass_g` | number |   | Grams |
| `tails_mass_kg` | number |   | Kilograms |
| `recovered_au_g_measured` | number | ✓ | Measured gold mass, grams |
| `recovery_pct_estimated` | number | ✓ | 0–100 |
| `notes` | string |   | Operator notes |
| `media` | array of strings |   | `ipfs://...` photo/video CIDs |
| `run_cid` | string |   | IPFS CID of the full run JSON |
| `media_cids` | array of strings |   | Individual media file CIDs |
| `attestation_uid` | string \| null |   | Set after on-chain attestation |
| `schema_version` | string | ✓ | `mineral-intel-ultrasound-run-v1` |

### Feed-type enum

| Value | Meaning |
| --- | --- |
| `placer_slurry` | Alluvial material in water suspension |
| `hardrock_float` | Crushed rock in water suspension |
| `mixed` | Mix of placer + hardrock |
| `other` | Other / unknown |

### Tray-geometry enum

| Value | Phase | Description |
| --- | --- | --- |
| `mini_duke_elutriator_v1` | 2 (May 20+) | Mini-Duke elutriation column baseline |
| `grooved_stainless_v1` | 3 (late June+) | Grooved stainless tray + ultrasound transducer |
| `other` | future | Reserved |

### Example record

```json
{
  "run_id": "2026-06-12-001",
  "date_utc": "2026-06-12T15:42:00Z",
  "site_lat": 48.7,
  "site_lon": -120.4,
  "feed_type": "placer_slurry",
  "feed_mass_kg": 12.4,
  "ultrasound_frequency_khz": 0,
  "tray_geometry": "mini_duke_elutriator_v1",
  "tray_angle_deg": 6,
  "run_duration_min": 22,
  "recovered_au_g_measured": 1.2,
  "recovery_pct_estimated": 71,
  "run_cid": "ipfs://Qm...",
  "attestation_uid": "0x...",
  "schema_version": "mineral-intel-ultrasound-run-v1"
}
```

---

## Constraints & validation

### Benchmark

- `commodity` must match the attested set
- `publication_date` ≥ `2020-01`
- All numeric metrics ≥ 0
- CIDs use the `ipfs://` prefix

### Run

- `run_id` matches `^\d{4}-\d{2}-\d{2}-\d{3}$`
- `date_utc` is valid ISO 8601 in UTC, ≤ current time
- `site_lat` ∈ [-90, 90]; `site_lon` ∈ [-180, 180]; both rounded ±0.1°
- `recovery_pct_estimated` ∈ [0, 100]
- `ultrasound_frequency_khz` ≥ 0
- `tray_angle_deg` ∈ [0, 90]

---

## IPFS layout

### Benchmark

```
source_cid   →  USGS MCS PDF (bafkrei...)
result_cid   →  Normalised JSON (Qm...)
```

### Run

```
run_cid      →  Full run JSON (Qm...)
media_cids[] →  Individual photos / video (Qm...)
```

Resolve via any IPFS gateway: `https://gateway.pinata.cloud/ipfs/<cid>`, `https://dweb.link/ipfs/<cid>`, `https://cloudflare-ipfs.com/ipfs/<cid>`.

---

## Version lock

Both schemas are **immutable v1**. Field semantics, ordering, and types do not change. New fields ship in v2 with a new UID; v1 remains valid for backwards compatibility.

### v2 candidates (non-binding)

- More commodities (rare earths, antimony, gallium, germanium, tungsten, tin, niobium, tantalum, aluminum, zinc, lead)
- Run logs: grain-size distribution, slurry density, particle-settling time, equipment maintenance lineage
- Benchmark: per-mine cost curves, scope-1/2 emissions per ton

---

## See also

- [API-REFERENCE.md](./API-REFERENCE.md) — endpoint shapes
- [VERIFICATION.md](./VERIFICATION.md) — verify on Base easscan
- [GETTING-STARTED.md](./GETTING-STARTED.md) — first-time walkthrough
