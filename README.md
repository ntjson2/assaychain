# Mineral Intelligence Benchmark API

**Attested USGS critical-mineral benchmarks + first-party gravity-separation run logs, served over x402 with on-chain provenance.**

- **Live endpoint:** `https://assaychain.com/api` (also resolves on `.net`)
- **Payment model:** [x402](https://x402.org) — USDC on Base mainnet
- **Provenance:** every paid response carries an IPFS CID + EAS attestation UID
- **Schemas:** `mineral-intel-benchmark-v1` and `mineral-intel-ultrasound-run-v1` are registered on Base and locked (new fields → v2)

---

## Quick start

### Free probes

```bash
# Service health + endpoint inventory
curl https://assaychain.com/api/health

# MCP tools/list (no payment)
curl -X POST https://assaychain.com/api/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

### Gated probe (returns 402)

```bash
curl -i https://assaychain.com/api/benchmark/copper
```

Response: `402 Payment Required` with `accepts[]` listing exact x402 `PaymentRequirements` (network, asset, amount, recipient).

### Paid call

Use any x402-aware client (e.g. [`x402-fetch`](https://github.com/coinbase/x402), [`x402-axios`](https://github.com/coinbase/x402)). Or use an MCP host that wires x402 for you (Claude Desktop via [`mcp-remote`](https://github.com/geelen/mcp-remote), Cursor, Smithery CLI).

---

## What's available

### Benchmark endpoints — $0.10 USDC flat

| Commodity | Endpoint | Data served | On-chain attestation |
| --- | --- | --- | --- |
| Copper | `GET /api/benchmark/copper` | ✓ live | ✓ |
| Gold | `GET /api/benchmark/gold` | ✓ live | ✓ |
| Silver | `GET /api/benchmark/silver` | ✓ live | ✓ |
| Cobalt | `GET /api/benchmark/cobalt` | rolling out | ✓ |
| Graphite | `GET /api/benchmark/graphite` | rolling out | ✓ |
| Lithium | `GET /api/benchmark/lithium` | rolling out | ✓ |
| Manganese | `GET /api/benchmark/manganese` | rolling out | ✓ |
| Nickel | `GET /api/benchmark/nickel` | rolling out | ✓ |

All 8 commodities are attested on Base. The MCP tool `get_commodity_benchmark` enumerates all 8; the REST surface adds the remaining 5 as data ships.

### Run-log endpoint — $0.10 USDC flat

```
GET /api/benchmark/ultrasound-grooved-tray
```

Founder-operated gravity-separation runs (mini-Duke elutriation in Phase 2; ultrasound transducer added in Phase 3). Filter via `feed_type`, `frequency_khz`, `min_recovery_pct`, `attested_only`, `limit`, `offset`.

### Structured-extract API (beta) — dynamic x402 ladder

| Endpoint | Method | Payment | Purpose |
| --- | --- | --- | --- |
| `/api/extract/estimate` | POST | free | Page/row probe + price quote + grade floor |
| `/api/extract/run` | POST | x402 (ladder: 0.10 / 0.50 / 1.00 / 2.50 / 5.00 USDC) | Run Gemini-2.5-Flash extraction |
| `/api/extract/result/:id` | GET | free (HMAC token) | Fetch result (24h TTL) |

Supports `pdf_digital`, `pdf_scanned`, `image`, `csv`, `txt`. `xlsx` / `docx` are 415-gated. Grade D results trigger an 80% auto-refund.

### MCP endpoint — `POST /api/mcp`

Streamable HTTP, stateless, [`@modelcontextprotocol/sdk`](https://github.com/modelcontextprotocol/typescript-sdk). Six tools:

| Tool | Cost | Purpose |
| --- | --- | --- |
| `ask_sales_agent` | free | Natural-language Q&A about the API (Groq-backed) |
| `get_commodity_benchmark` | $0.10 | Pull benchmark for one of 8 commodities |
| `get_ultrasound_run_data` | $0.10 | Query run logs |
| `estimate_extract` | free | Quote a paid extraction job |
| `run_extract` | discovery only | Returns the paid endpoint + price (MCP can't carry the `X-PAYMENT` header) |
| `get_extract_result` | free (token) | Fetch a completed extraction |

---

## Verification

Every paid response includes a `provenance` block:

```json
"provenance": {
  "attestation_uid": "0x5546f1ab9227f9d73d359ee6065c144c153390627019672e5ac6e774330fbd38",
  "source_cid": "ipfs://bafkrei...",
  "result_cid": "ipfs://Qm...",
  "verify_url": "https://base.easscan.org/attestation/view/0x5546..."
}
```

Open `verify_url` to inspect attestor, schema, timestamp, and data on Base.

### Schema UIDs (immutable v1)

| Schema | UID |
| --- | --- |
| `mineral-intel-benchmark-v1` | `0xde9e029944dd16f3dad09458ec50b1823e4fbeaa35e6779509aeb84059bb8531` |
| `mineral-intel-ultrasound-run-v1` | `0x7fe15e2b15e89e5e16348683b62c90c9139fdbf1b24d424d7931b4572334d73d` |

Browse: `https://base.easscan.org/schema/view/<schema-uid>`.

---

## Integration

### MCP host (Claude Desktop, Cursor, etc.)

The server speaks Streamable HTTP at `POST /api/mcp`. For hosts that only spawn stdio MCP servers, bridge with [`mcp-remote`](https://github.com/geelen/mcp-remote):

```json
{
  "mcpServers": {
    "mineral-intel": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://assaychain.com/api/mcp"
      ]
    }
  }
}
```

### Direct HTTP

Base URL: `https://assaychain.com/api`

All responses are JSON. See [API-REFERENCE.md](./API-REFERENCE.md).

### Registries

Discovery surfaces (submissions in progress): [x402.org/ecosystem](https://x402.org/ecosystem), [Smithery](https://smithery.ai), [Glama](https://glama.ai), [mcp.so](https://mcp.so), [PulseMCP](https://www.pulsemcp.com).

---

## Documentation

- [GETTING-STARTED.md](./GETTING-STARTED.md) — first-time walkthrough
- [API-REFERENCE.md](./API-REFERENCE.md) — every endpoint, exact response shapes
- [SCHEMA.md](./SCHEMA.md) — v1 schema fields (locked)
- [EXAMPLES.md](./EXAMPLES.md) — cURL / JS / Python
- [VERIFICATION.md](./VERIFICATION.md) — verify attestations on Base
- [x402-GUIDE.md](./x402-GUIDE.md) — payment protocol
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — common errors
- [FAQ.md](./FAQ.md) — frequently asked questions
- [LICENSE.md](./LICENSE.md) — MIT (code/docs) + CC BY 4.0 (run-log data) + USGS public domain (benchmarks)

---

## Acknowledgments

- **Benchmark source:** USGS Mineral Commodity Summaries 2025/2026
- **Chain:** [Base](https://base.org)
- **Attestation:** [Ethereum Attestation Service](https://attest.org)
- **Pinning:** [Pinata](https://pinata.cloud)
- **Payment protocol:** [x402](https://x402.org) (Coinbase)
- **Extraction model:** Google Gemini 2.5 Flash

---

## License

MIT — see [LICENSE.md](./LICENSE.md). USGS data is public domain; first-party run-log data is CC BY 4.0.

---

## Support

Issues and feedback: open an issue on the public GitHub repo. This is a solo-maintained project on a field-season schedule; responses may be slow May–September.

_Last updated: 2026-05-15._
