# Frequently Asked Questions

---

## General

**Q: What is this API?**
A: Attested USGS mineral benchmark data + founder-operated gravity separation run logs, published on-chain via EAS on Base. Every response includes IPFS CID + attestation UID for provenance.

**Q: Who runs this?**
A: Quetzal Collective LLC (solo founder). Data is from USGS (benchmarks) and original field experiments (run logs).

**Q: Is this real or fake data?**
A: Real. Benchmarks are from USGS Mineral Commodity Summaries. Run logs are from actual gravity separation experiments. Every record is attested on-chain.

**Q: Can I use this commercially?**
A: Yes. Each call costs $0.10 USDC. USGS data is public domain. Run logs are first-party data (licensed for use).

---

## Payment

**Q: How much does each call cost?**
A: $0.10 USDC on Base. That's it. No subscriptions, no hidden fees.

**Q: Can I make free calls?**
A: Yes. `/api/health` is free. All benchmark and run endpoints require payment.

**Q: Do I have to pay every time I call the same endpoint?**
A: Yes. Each call = $0.10 USDC. Caching (1 hour for benchmarks, 5 minutes for runs) means repeated calls may return cached data at the gateway level.

**Q: What if I don't want to pay?**
A: You can call `/api/health` for free, or read the [Data Schema](./SCHEMA.md) documentation.

**Q: What if the payment fails?**
A: You get a 402 response. No charge. Try again.

**Q: Can I negotiate volume pricing?**
A: Not yet. Post an issue on GitHub or reach out.

**Q: Why is the payment required?**
A: To sustain the infrastructure, IPFS pinning, EAS attestations, and ongoing data curation. This covers server costs, IPFS storage, and gas fees.

---

## Data

**Q: Where does the benchmark data come from?**
A: USGS Mineral Commodity Summaries (annual publication). Data is public domain.

**Q: How often is benchmark data updated?**
A: USGS publishes Mineral Commodity Summaries annually. The API mirrors the latest MCS edition; `data.source` and `data.publication_date` show which edition.

**Q: Where does the run log data come from?**
A: Founder's own gravity separation experiments. Recorded in real-time, attested on-chain.

**Q: How many run logs are there?**
A: Growing. See the runs endpoint for current count. Started May 20, 2026.

**Q: Can I use run log data for my own research?**
A: Yes. It's publicly available and attested on-chain. Cite the attestation UID.

**Q: Is the run log data peer-reviewed?**
A: No. It's first-party experimental data. Verify the attestation on easscan.org if needed.

**Q: Can the data be deleted?**
A: No. Attestations are immutable on Base. Data is pinned to IPFS indefinitely.

---

## Attestation & Verification

**Q: What does "attestation" mean?**
A: A cryptographic record on the Base blockchain proving the data exists, hasn't been modified, and was recorded at a specific time.

**Q: How do I verify an attestation?**
A: Copy the `attestation_uid` from the API response, go to [base.easscan.org](https://base.easscan.org), and search it.

**Q: Who signed the attestation?**
A: Inspect the attestation on [base.easscan.org](https://base.easscan.org) — the `attestor` field shows the signing wallet. Each paid response also returns a `verify_url` pointing directly at that record.

**Q: Can the attestation be revoked?**
A: No. Benchmark schema is "immutable." Run log schema is also immutable.

**Q: What's the difference between attestation and payment receipt?**
A: Attestation proves the data is real and hasn't been modified. Payment receipt proves you paid for the call. Both are important but separate.

**Q: Can I use this in legal proceedings?**
A: Attestations are immutable, on-chain records. Consult your legal team about admissibility in your jurisdiction.

---

## Technical

**Q: What's the API response format?**
A: JSON. See [API Reference](./API-REFERENCE.md) for full specs.

**Q: Can I use this API in my app?**
A: Yes. It's public and REST-ful. See [Code Examples](./EXAMPLES.md).

**Q: Is there an SDK?**
A: Not bespoke. The REST surface is plain JSON over HTTPS — use any x402 client ([`x402-fetch`](https://github.com/coinbase/x402), [`x402-axios`](https://github.com/coinbase/x402)). MCP clients can connect via [`mcp-remote`](https://github.com/geelen/mcp-remote).

**Q: What programming languages are supported?**
A: Any language that can make HTTP requests (cURL, JavaScript, Python, Go, etc.).

**Q: Can I cache the responses?**
A: Benchmark responses ship with `Cache-Control: public, max-age=3600` (1 h). Run-log and extract responses are `no-store` (always live).

**Q: Is there a GraphQL API?**
A: No. REST only (for now).

**Q: Can I subscribe to updates?**
A: No webhooks. Poll the run endpoint to check for new runs.

**Q: What's the rate limit?**
A: None per se. You're rate-limited by payment — each call costs $0.10 USDC.

**Q: Is this API GDPR-compliant?**
A: Benchmark data is anonymous aggregates. Run data is anonymized (coordinates rounded ±0.1°). No PII is collected or stored.

---

## Downtime & Reliability

**Q: What's the SLA (Service Level Agreement)?**
A: Best-effort. No SLA currently. This is a solo-founder project.

**Q: What if the API goes down?**
A: Try `GET /api/health` first. If the health check is failing, open an issue on the GitHub repo.

**Q: Can I host a copy myself?**
A: Not yet. The code isn't open-source. Benchmarks and runs are on IPFS/EAS (decentralized) so data persists.

**Q: What if you shut it down?**
A: Data is immutable on Base and pinned to IPFS. The attestations and CIDs will survive. But the `/api` endpoint would be gone. Consider archiving important attestation UIDs.

---

## Business

**Q: Is this project for sale?**
A: Not currently. The project is solo-founder-operated.

**Q: Can I integrate this into my product?**
A: Yes. Pay per call as you use it. No licensing agreement needed.

**Q: Do you offer white-label or custom versions?**
A: Not yet. File an issue if interested.

**Q: What's your business model?**
A: Revenue per API call ($0.10 USDC). Long-term: dataset licensing + consulting.

**Q: Why is this so cheap?**
A: To validate market demand. Pricing may increase if demand justifies it.

---

## Data Roadmap

**Q: What commodities will be added?**
A: Phase 1: 8 critical minerals (copper, gold, silver, lithium, cobalt, nickel, manganese, graphite).
Phase 2+: More USGS commodities on demand.

**Q: Will you add real-time price data?**
A: Maybe. Requires integrating with commodity exchanges (Comex, LME, etc.). Not in current roadmap.

**Q: Will you add mining company data?**
A: Maybe. Requires partnerships or scraping. Not in current roadmap.

**Q: Will you add environmental/ESG data?**
A: Maybe. Carbon footprint and energy intensity are already in the schema. Scope-3 data (supply chain) is out-of-scope for now.

**Q: How long will you keep run log data?**
A: Indefinitely. IPFS pinning is permanent. EAS attestations are forever. No planned deprecation.

---

## Support

**Q: How do I report a bug?**
A: File an issue on GitHub with:
- Exact error message
- Endpoint called
- Request/response headers
- Timestamp

**Q: How do I suggest a feature?**
A: File an issue with `[feature-request]` label.

**Q: How do I get help?**
A: Check [Troubleshooting](./TROUBLESHOOTING.md) first. Then:
- [Getting Started](./GETTING-STARTED.md) — walkthrough
- [API Reference](./API-REFERENCE.md) — endpoint docs
- [Code Examples](./EXAMPLES.md) — working samples

**Q: How do I contact the team?**
A: File an issue on GitHub. This is a solo project, so responses may be slow during field season (May–September).

---

## See Also

- [Getting Started](./GETTING-STARTED.md) — First-time user guide
- [API Reference](./API-REFERENCE.md) — Complete endpoint documentation
- [Troubleshooting](./TROUBLESHOOTING.md) — Common errors and fixes
- [Verification Guide](./VERIFICATION.md) — How to verify attestations
