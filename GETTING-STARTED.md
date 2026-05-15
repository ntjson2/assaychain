# Getting Started

Step-by-step guide to using the Mineral Intelligence Benchmark API.

---

## Table of Contents

1. [Test the free endpoint](#test-the-free-endpoint)
2. [Make a paid call](#make-a-paid-call)
3. [Query run data](#query-run-data)
4. [Verify attestations](#verify-attestations)
5. [Use with an AI agent](#use-with-an-ai-agent)
6. [Next steps](#next-steps)

---

## Test the Free Endpoint

Start here — no setup required, no payment.

```bash
curl https://us-central1-fourmonth-73efe.cloudfunctions.net/api/health
```

(The vanity host `https://assaychain.com` also resolves to the same backend.)

You should see:

```json
{
  "status": "ok",
  "service": "mineral-intel-benchmark",
  "version": "1.0.0",
  "endpoints": { ... },
  "chain": "base",
  "timestamp": "2026-05-15T18:30:00Z"
}
```

This confirms the API is running.

---

## Make a Paid Call

### Step 1: Understand x402

The API uses **x402**, a micropayment protocol for APIs. Here's what happens:

1. You call an endpoint (e.g., `/api/benchmark/copper`)
2. You get back `402 Payment Required` with a price
3. You settle the payment on Base (USDC)
4. The same call returns data

### Step 2: Use an x402-aware client

To make paid calls, use:
- **Claude Desktop** (built-in x402)
- **Cursor IDE** (built-in x402)
- **MCP client** with x402 support
- **Manual curl** (if you know the payment protocol)

### Step 3: Make the call

#### Option A: Claude Desktop (via `mcp-remote` bridge)

Claude Desktop spawns stdio MCP servers. To reach our Streamable HTTP endpoint, bridge with [`mcp-remote`](https://github.com/geelen/mcp-remote):

1. Open the Claude Desktop config file:
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%/Claude/claude_desktop_config.json`
2. Add the server:
   ```json
   {
     "mcpServers": {
       "mineral-intel": {
         "command": "npx",
         "args": [
           "-y",
           "mcp-remote",
           "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/mcp"
         ]
       }
     }
   }
   ```
3. Restart Claude Desktop.
4. Ask the model: *"Use the `ask_sales_agent` tool to summarise this API"*, or *"Call `get_commodity_benchmark` for copper."*

Note: MCP itself does **not** carry the x402 `X-PAYMENT` header. Paid tools return the 402 envelope; settle payment with an x402 client (e.g. `x402-fetch`) and POST to the REST endpoint directly. The MCP layer is best used for free discovery (`ask_sales_agent`, `estimate_extract`, `get_extract_result`).

#### Option B: Direct HTTP with an x402 client

```bash
# Probe — returns 402 with the canonical accepts[] PaymentRequirements
curl -i https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper

# Paid call — header value is the base64-encoded x402 payload your client builds
curl -H "X-PAYMENT: <base64 x402 payload>" \
  https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper
```

Use [`x402-fetch`](https://github.com/coinbase/x402) or [`x402-axios`](https://github.com/coinbase/x402) to build the `X-PAYMENT` header. See [x402-GUIDE.md](./x402-GUIDE.md) for details.

### Step 4: Parse the response

You'll get back USGS copper data wrapped with a `provenance` envelope. Trimmed shape:

```json
{
  "ok": true,
  "data": {
    "commodity": "copper",
    "symbol": "Cu",
    "unit": "metric ton",
    "source": "USGS Mineral Commodity Summaries 2026",
    "price": { "spot_usd_per_lb": 4.2 },
    "production": { "world_mine_output_2025_ktonnes": 22000 },
    "recovery_benchmarks": { "flotation_pct": { "typical": 87 } },
    "attestation_uid": "0x5546f1ab9227...",
    "source_cid": "ipfs://bafkrei...",
    "result_cid": "ipfs://Qm..."
  },
  "provenance": {
    "attestation_uid": "0x5546f1ab9227...",
    "verify_url": "https://base.easscan.org/attestation/view/0x5546..."
  },
  "meta": { "schema_version": "mineral-intel-benchmark-v1", "generated_at": "2026-05-15T18:30:00Z" }
}
```

See [API-REFERENCE.md](./API-REFERENCE.md) for the full payload.

---

## Query Run Data

The API also serves founder-operated gravity separation run logs.

### Endpoint

```
GET /api/benchmark/ultrasound-grooved-tray
```

Same $0.10 USDC payment model.

### Simple query

Get the first 10 runs (unpaid call returns 402 with the accepts[] envelope):

```bash
curl -i "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/ultrasound-grooved-tray?limit=10"
```

### Filtered query

Placer-slurry runs with ≥70% recovery, attested-only:

```bash
curl -i "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/ultrasound-grooved-tray?feed_type=placer_slurry&min_recovery_pct=70&attested_only=true"
```

Response:

```json
{
  "ok": true,
  "device": "ultrasound-grooved-tray",
  "stats": { "total_runs": 42, "attested_runs": 38, "avg_recovery_pct": 64.3, "best_recovery_pct": 89 },
  "pagination": { "offset": 0, "limit": 10, "returned": 10, "total": 42 },
  "runs": [
    {
      "run_id": "2026-06-12-001",
      "date_utc": "2026-06-12T15:42:00Z",
      "feed_type": "placer_slurry",
      "ultrasound_frequency_khz": 0,
      "recovery_pct_estimated": 71,
      "recovered_au_g_measured": 1.2,
      "attestation_uid": "0x..."
    }
  ]
}
```

See [API Reference](./API-REFERENCE.md) for all query parameters.

---

## Verify Attestations

Every response includes an `attestation_uid`. This proves the data is:
- Authentic (signed by the data provider)
- Immutable (recorded on-chain)
- Timestamped (exact moment of attestation)

### Verify a benchmark attestation

1. Copy the `attestation_uid` from the response
2. Visit [base.easscan.org](https://base.easscan.org)
3. Paste the UID in the search box
4. Inspect:
   - Attestor address
   - IPFS CID of the data
   - On-chain timestamp
   - Full attestation record

Example UID:
```
0xde9e029944dd16f3dad09458ec50b1823e4fbeaa35e6779509aeb84059bb8531
```

### Verify a run log attestation

Same process. Run log attestations are attested under the schema:
```
0x7fe15e2b15e89e5e16348683b62c90c9139fdbf1b24d424d7931b4572334d73d
```

See [Verification Guide](./VERIFICATION.md) for more details.

---

## Use with an AI Agent

### Claude Desktop

Same config as Option A above — use `mcp-remote` to bridge to the Streamable HTTP endpoint:

```json
{
  "mcpServers": {
    "mineral-intel-benchmark": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://us-central1-fourmonth-73efe.cloudfunctions.net/api/mcp"
      ]
    }
  }
}
```

Use the free `ask_sales_agent` tool for natural-language Q&A. For paid tools, settle x402 separately with an x402-fetch client against the REST endpoint.

### Cursor IDE

Similar setup — add the MCP server to Cursor's config and use in the editor's AI chat.

### Custom agent

Use `x402-fetch` directly:

```javascript
import { wrapFetchWithPayment } from 'x402-fetch';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);
const fetchPaid = wrapFetchWithPayment(fetch, account);

const res = await fetchPaid('https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper');
const json = await res.json();
console.log(json.data, json.provenance);
```

See [Code Examples](./EXAMPLES.md) for full implementations.

---

## Next Steps

1. **Explore the data** — Query different commodities and runs
2. **Verify attestations** — Check the on-chain records
3. **Build an integration** — Use in your app or agent
4. **Give feedback** — Issues and suggestions welcome

---

## Common Questions

**Q: Do I need to pay every time?**
A: Yes, each call costs $0.10 USDC. But x402 clients (Claude Desktop, Cursor) handle payment automatically.

**Q: What if the payment fails?**
A: You get a 402 response with error details. See [Troubleshooting](./TROUBLESHOOTING.md).

**Q: Can I cache results?**
A: Benchmark responses are sent with `Cache-Control: public, max-age=3600`. Run-log responses are `no-store` (always live).

**Q: How do I know the data is real?**
A: Every response includes an `attestation_uid`. Verify it on [base.easscan.org](https://base.easscan.org) — it's immutable, on-chain proof.

**Q: What's the difference between benchmarks and runs?**
A: Benchmarks are USGS aggregate data (grade cutoffs, spot pricing, recovery benchmarks, production stats). Runs are first-party empirical data from the founder's gravity-separation rig.

---

## Troubleshooting

If something goes wrong, see [Troubleshooting](./TROUBLESHOOTING.md).

If your question isn't answered, see [FAQ](./FAQ.md).
