# x402 Payment Guide

Understanding the x402 micropayment protocol used by this API.

---

## What is x402?

x402 is a micropayment protocol for APIs that:

- **Settles instantly** on blockchain (~3–4 seconds on Base)
- **Requires no subscription** — pay per call
- **Works with multiple clients** — Claude Desktop, Cursor, custom agents
- **Supports USDC** — stablecoin payments, no volatility
- **Non-reversible** — once paid, payment cannot be disputed

Official: [x402.org](https://x402.org) | Spec: [github.com/coinbase/x402](https://github.com/coinbase/x402)

---

## How Payments Work

### For the user (you)

1. You call an endpoint without payment → get `402 Payment Required` with an `accepts[]` array of PaymentRequirements (scheme, network, asset, payTo, maxAmountRequired)
2. Your x402 client builds a signed payload and sets the `X-PAYMENT` request header
3. You call again — the server validates the payload, settles via the x402 facilitator, and returns the data

### For the server (us)

1. Receive call → check for `X-PAYMENT` header
2. Validate the payload and settle via the x402 facilitator on Base
3. Return data (200 OK)

---

## Payment Flow

```
┌─────────────────────────────────────────────┐
│ 1. Call endpoint without payment            │
│    GET /api/benchmark/copper                │
└──────────────────┬──────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────┐
│ 2. Receive 402 with pricing info            │
│ {                                           │
│   "error": "Payment required",              │
│   "price": "0.10",                          │
│   "currency": "USDC",                       │
│   "network": "base",                        │
│   "recipient": "0x750977..."                │
│ }                                           │
└──────────────────┬──────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────┐
│ 3. Settle payment on Base                   │
│    Send 0.10 USDC to recipient address      │
└──────────────────┬──────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────┐
│ 4. Call again with payment header           │
│    GET /api/benchmark/copper                │
│    Header: X-PAYMENT: <signature>        │
└──────────────────┬──────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────┐
│ 5. Receive data (200 OK)                    │
│ {                                           │
│   "ok": true,                               │
│   "data": { "commodity": "copper", ... },   │
│   "provenance": { "attestation_uid": ... }, │
│   "meta": { "schema_version": "...v1" }     │
│ }                                           │
└─────────────────────────────────────────────┘
```

---

## Pricing

| Endpoint | Price | Currency | Network |
|---|---|---|---|
| `GET /api/benchmark/:commodity` | $0.10 | USDC | Base |
| `GET /api/benchmark/ultrasound-grooved-tray` | $0.10 | USDC | Base |
| `POST /api/extract/estimate` | Free | — | — |
| `POST /api/extract/run` | $0.10–$5.00 (ladder) | USDC | Base |
| `GET /api/extract/result/:id` | Free (HMAC token) | — | — |
| `GET /api/health`, `POST /api/mcp` (free tools) | Free | — | — |

One payment = one call. Pricing is set by the `x402-next` middleware; integer base units are sent in the 402 envelope so clients never need to format decimals.

---

## Making Payments

### Option 1: Free MCP tools via Claude Desktop

Claude Desktop can connect to our MCP endpoint through the [`mcp-remote`](https://github.com/geelen/mcp-remote) bridge and use the **free** tools (`ask_sales_agent`, `estimate_extract`, `get_extract_result`) directly.

1. Add the API as an MCP server (see [Getting Started](./GETTING-STARTED.md))
2. Ask Claude to call `ask_sales_agent` or another free tool
3. You get the data

**Important:** MCP itself does not carry the `X-PAYMENT` header. Paid REST endpoints must be called by an x402-aware HTTP client (Option 2). Inside MCP, paid tools like `get_commodity_benchmark` will return the 402 envelope so the user can settle outside the MCP transport.

### Option 2: Direct HTTP with an x402 client

If you're building your own integration, use [`x402-fetch`](https://github.com/coinbase/x402) (JS) or `x402-axios` to do the heavy lifting:

```javascript
import { wrapFetchWithPayment } from 'x402-fetch';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);
const fetchPaid = wrapFetchWithPayment(fetch, account);

const res = await fetchPaid(
  'https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper'
);
const json = await res.json();
```

Under the hood, the client:

1. Probes the endpoint, reads `accepts[]` from the 402 envelope.
2. Signs an EIP-712 USDC transfer authorization to the `payTo` address for `maxAmountRequired` base units (e.g. `100000` = $0.10).
3. Base64-encodes the payload and retries with `X-PAYMENT: <payload>`.
4. The server settles via the x402 facilitator and returns the data.

You never hand-sign transactions or call basescan — the client does it.

**Recipient and asset are fixed:**

- Network: Base mainnet
- Asset: USDC `0x833589fCD6eDb6E08f4c7C32D4f71b3566dA4ef5`
- Recipient: `0x750977976Ab85A4Ce5AAbb2e1a9fc80a633f2769`
- Amount: `100000` base units ($0.10) per call

---

## Payment Receipts

On-chain settlement is the receipt. After a paid call:

- The USDC transfer to `0x750977976Ab85A4Ce5AAbb2e1a9fc80a633f2769` is visible on [basescan.org](https://basescan.org)
- The response itself carries the EAS `provenance.attestation_uid` and `provenance.verify_url`, anchoring the *data* to a permanent on-chain record

A structured `PAYMENT-RESPONSE` HTTP header carrying the settlement tx hash is on the roadmap but not currently emitted. For now, reconcile via basescan or via your x402 client's settlement log.

---

## Supported Networks

- **Base mainnet** (current)
  - USDC: `0x833589fCD6eDb6E08f4c7C32D4f71b3566dA4ef5`
  - Recipient: `0x750977976Ab85A4Ce5AAbb2e1a9fc80a633f2769`

Base was chosen because:
- Fast (~3–4 second confirmation)
- Cheap gas (fractions of a cent)
- USDC is stable
- x402 spec-compliant

---

## Batch & Subscription Pricing

Currently: **pay-per-call** only ($0.10 per call).

Future options (not yet implemented):
- **Batch pricing** — 10+ calls = lower per-call rate
- **Subscription** — unlimited calls for X USDC/month
- **Volume tiers** — enterprise discounts

If interested in bulk access, reach out.

---

## Refunds & Disputes

**No refunds or disputes.** x402 payments are **non-reversible by design**.

This is a feature, not a bug:
- Server cannot reverse your payment and give you bad data
- You cannot reverse your payment and claim you didn't receive data
- Both parties are protected by the immutability of the payment

If you believe an error occurred, file an issue on GitHub instead.

---

## Rate Limiting

No rate limiting per se, but:

- Each x402 payment grants one call
- To call again, you must pay again
- No "unlimited calls for 24 hours" model

---

## Cost Examples

| Scenario | Calls | Cost | Notes |
|---|---|---|---|
| Try all 8 commodities | 8 | $0.80 | One benchmark each |
| Query run data once | 1 | $0.10 | Up to 100 results per page |
| Monthly explorer | 30 | $3.00 | One call per day |
| Weekly researcher | 100 | $10.00 | ~14 calls/week |
| Production integration | 10,000 | $1,000 | 100 calls/day, 1 year |

---

## Troubleshooting Payments

### Problem: "402 Payment Required" every time

This means the payment header is missing or invalid. Solutions:

1. **If using Claude Desktop** — restart Claude, re-add the MCP server
2. **If using manual payments** — regenerate your signature, check the timestamp
3. **If gas failed** — check your Base wallet has USDC balance

### Problem: "Invalid payment signature"

The signature doesn't match the request. Check:

- Endpoint URL is correct
- Amount ($0.10 USDC) is correct
- Recipient address is correct
- Timestamp is fresh (not stale)
- Signature was computed correctly

### Problem: "Payment not found on-chain"

The payment was initiated but hasn't been confirmed yet. Wait 5–10 seconds and retry.

See [Troubleshooting](./TROUBLESHOOTING.md) for more.

---

## Security

### Your private key

- **Never** share your private key
- **Never** commit it to GitHub
- **Never** use a key you've exposed anywhere
- Store safely offline or in a secure wallet

### Server-side validation

The server validates:
- Payment header is present
- Signature matches the request
- Payment is on-chain and confirmed
- Amount is correct ($0.10)
- Recipient is correct

### Why Base?

Base is used because:
- **Finality** — no reorg risk after ~13 seconds
- **Throughput** — supports high call volumes
- **Cost** — gas fees <$0.001 per call
- **USDC** — stablecoin, no volatility

---

## Next Steps

1. **Set up Claude Desktop** (easiest)
2. **Make your first call** — see [Getting Started](./GETTING-STARTED.md)
3. **Verify payment** — check your Base wallet on basescan.org
4. **Use the data** — build your integration

---

## Learn More

- [x402.org](https://x402.org) — Official protocol
- [x402 spec](https://github.com/coinbase/x402) — Technical details
- [x402-fetch SDK](https://github.com/coinbase/x402-fetch) — JavaScript client library
- [Base docs](https://docs.base.org) — Base blockchain
- [USDC docs](https://www.circle.com/en/usdc) — USDC stablecoin

---

## Questions?

See [FAQ](./FAQ.md) or [Troubleshooting](./TROUBLESHOOTING.md).
