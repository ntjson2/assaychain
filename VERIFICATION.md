# Verification Guide

How to verify that data from the API is authentic, immutable, and tamper-proof.

---

## What Gets Attested?

Every response from the API includes an **attestation UID**. This means:

1. A cryptographic signature was recorded on-chain (Base mainnet)
2. The data is immutable — it cannot be changed
3. The timestamp is permanent — you know exactly when it was attested
4. The IPFS CID is linked — you can fetch the exact data that was signed

---

## Verify a Benchmark Attestation

### Step 1: Get the UID from a response

```bash
curl https://assaychain.com/api/benchmark/copper \
  -H "X-PAYMENT: <your-signature>"
```

Response includes a `provenance` block:

```json
{
  "ok": true,
  "data": { "commodity": "copper", "attestation_uid": "0x5546f1ab9227..." },
  "provenance": {
    "attestation_uid": "0x5546f1ab9227...",
    "verify_url": "https://base.easscan.org/attestation/view/0x5546f1ab9227..."
  }
}
```

The quickest path: open the `verify_url` directly in your browser.

### Step 2: Open on easscan

Visit `https://base.easscan.org/attestation/view/<attestation_uid>` directly, or paste the UID into the search bar at [base.easscan.org](https://base.easscan.org).

### Step 3: Inspect the record

You'll see:

| Field | Description |
|---|---|
| **Attestor** | Founder's signing wallet — inspect the live record; the address shown is canonical |
| **Schema** | `0xde9e029944dd16f3dad09458ec50b1823e4fbeaa35e6779509aeb84059bb8531` (`mineral-intel-benchmark-v1`) |
| **Timestamp** | Exact UTC moment the attestation was recorded |
| **Data** | Encoded subset: commodity, year, schema_version, source_url, source_cid, result_cid, recovery_pct_typical |
| **IPFS CIDs** | `source_cid` (USGS PDF) and `result_cid` (normalized JSON) |
| **Revocable** | No — attestations are permanent |

---

## Verify a Run Log Attestation

Same process, but the **schema UID** is:

```
0x7fe15e2b15e89e5e16348683b62c90c9139fdbf1b24d424d7931b4572334d73d
```

Individual run attestations have their own per-record UIDs, returned in each run's `attestation_uid` field. Use `https://base.easscan.org/attestation/view/<uid>` to inspect any single run.

---

## What's Being Attested?

### Benchmarks

The on-chain attestation encodes the canonical subset:

- `commodity` (string)
- `year` (uint16)
- `schema_version` (string)
- `source_url` (string — USGS landing page)
- `source_cid` (string — IPFS CID of source PDF)
- `result_cid` (string — IPFS CID of normalized JSON)
- `recovery_pct_typical` (uint16)

The full data tree (price, grade cutoffs, production stats, chemistry, recovery benchmarks) lives in the IPFS `result_cid` JSON, which is fetched and returned in the API's `data` field. The CID is what anchors the on-chain record — if a single byte of the result JSON changes, the CID changes and no longer matches the attestation.

### Run logs

The attestation includes:

- Run ID (e.g., `2026-06-12-001`)
- Date (UTC)
- Site coordinates (rounded ±0.1° for OPSEC)
- Feed type (placer slurry, hardrock float, etc.)
- Feed mass (kg)
- Ultrasound frequency (0 = elutriation only)
- Ultrasound power (watts)
- Tray geometry
- Run duration (minutes)
- Recovery % (estimated)
- Recovered gold mass (grams, measured)
- Timestamp

---

## Verify via IPFS

Every attestation includes an IPFS CID of the original data.

### Get the CID

From the API response:

```json
{
  "result_cid": "Qm...",
  "source_cid": "bafkrei..."
}
```

### Fetch via IPFS

```bash
# Result data (normalized JSON)
ipfs get Qm... > data.json

# Source data (USGS PDF for benchmarks)
ipfs get bafkrei... > source.pdf
```

Or use a public gateway:

```bash
# Via cloudflare IPFS gateway
curl https://cloudflare-ipfs.com/ipfs/Qm...

# Via dweb.link
curl https://dweb.link/ipfs/Qm...
```

---

## Chain of Custody

Here's the complete chain:

```
Original data (PDF/JSON)
    ↓
[Pin to IPFS via Pinata]
    ↓ (get IPFS CID)
Normalize to JSON
    ↓
[Attest on EAS/Base]
    ↓ (get attestation UID)
API response includes both CIDs + UID
    ↓
User verifies on easscan.org
    ↓
User confirms IPFS CID matches
    ↓
✓ Data is authentic and immutable
```

---

## Why This Matters

### For traders / processors

- Verify that benchmark data hasn't been manipulated
- Check the exact timestamp the data was recorded
- Ensure it's from a legitimate source (the attestor address)

### For regulators

- Audit trail of when data was published
- Immutable record on Base mainnet
- Source attribution (IPFS CID points to original)

### For the DAO / marketplace

- Proves first-party data hasn't been tampered with
- Enables reputation scoring (how many attested runs?)
- Creates permanent, verifiable dataset

---

## Verification Checklist

When using data from the API:

- [ ] Copy the `attestation_uid` from the response (or use the `verify_url`)
- [ ] Open it on [base.easscan.org](https://base.easscan.org)
- [ ] Confirm the schema UID matches the data type (benchmark vs. run-log)
- [ ] Note the attestor address and timestamp
- [ ] (Optional) Fetch the IPFS CID and verify the hash matches
- [ ] Use the data with confidence

---

## Frequently Asked Questions

**Q: Can the attestation be deleted?**
A: No. Base is immutable. The attestation is permanent.

**Q: Who signed this?**
A: The current attestor wallet is shown on every attestation record at [base.easscan.org](https://base.easscan.org). Inspect any record returned by the API to confirm the active signer.

**Q: Is the schema locked?**
A: Yes. Both v1 schemas are immutable and registered on Base.

**Q: How do I know the source data is real?**
A: For benchmarks, the source PDF is linked via IPFS CID and was published by USGS (Mineral Commodity Summaries). For runs, the founder's own data.

**Q: Can I use this in a legal contract?**
A: The attestation UID is immutable, on-chain proof. Consult your legal team about admissibility in your jurisdiction.

---

## Learn More

- [base.easscan.org](https://base.easscan.org) — EAS attestation explorer
- [EAS docs](https://docs.attest.sh) — Ethereum Attestation Service
- [IPFS docs](https://docs.ipfs.tech) — InterPlanetary File System
- [Base docs](https://docs.base.org) — Base blockchain

---

## Support

If you have questions about verification, see [FAQ](./FAQ.md) or file an issue on GitHub.
