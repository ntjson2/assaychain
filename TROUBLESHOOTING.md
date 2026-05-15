# Troubleshooting

Common errors and how to fix them.

---

## HTTP Status Codes

### 200 OK

✓ Success. You got the data.

### 402 Payment Required

Payment is needed to access this endpoint.

**Fix:**
1. Read the `accepts[]` array on the 402 response (scheme, network, asset, payTo, maxAmountRequired)
2. Use an x402 client (`x402-fetch`, `x402-axios`) to build a signed `X-PAYMENT` payload
3. Retry the request with the `X-PAYMENT` header set
4. See [x402 Guide](./x402-GUIDE.md)

**Still getting 402 after payment?**
- Wait 5–10 seconds for on-chain confirmation
- Check your Base wallet has sufficient USDC balance
- Verify the payment was sent to the correct recipient address
- Check network is Base mainnet (not testnet or another chain)

### 404 Not Found

Endpoint doesn't exist or typo in the path.

**Fix:**
1. Check the path spelling
2. Verify it's in the [API Reference](./API-REFERENCE.md)
3. Example: `/api/benchmark/copper` (not `/api/copper` or `/benchmark/copper`)
4. Call `/api/health` to see available endpoints

### 500 Internal Server Error

Server-side error.

**Fix:**
1. Retry after 10 seconds
2. Check if the endpoint is down (try `/api/health`)
3. File an issue on GitHub with:
   - Exact endpoint you called
   - Request headers
   - Response body
   - Timestamp

---

## Payment Issues

### "Payment required" even after I paid

**Cause:** Payment not yet on-chain or signature invalid.

**Fix:**
1. Wait 10 seconds for Base confirmation
2. Verify on-chain:
   ```bash
   # Check transaction on basescan.org
   # https://basescan.org/tx/<your-tx-hash>
   ```
3. Regenerate payment signature
4. Try again

### "Invalid payment signature"

**Cause:** `X-PAYMENT` header is malformed, expired, or doesn't match the request.

**Fix:**
1. Verify the `X-PAYMENT` header is present and non-empty
2. Regenerate via your x402 client (the payload is request-bound — don't reuse it across endpoints)
3. Ensure your client clock is reasonably accurate (within a few minutes of UTC)

### "Payment not found on-chain"

**Cause:** Payment was initiated but hasn't confirmed.

**Fix:**
1. Wait 10–20 seconds for confirmation
2. Check on basescan.org that the transaction is confirmed
3. Retry the API call

### Transaction cost too high

**Cause:** Base gas prices spiked.

**Note:** Gas is typically <$0.001. If higher:
1. Wait a few minutes
2. Retry when network is less congested

---

## Data Issues

### Empty response

**Endpoint:** `/api/benchmark/ultrasound-grooved-tray`

**Cause:** No runs match your filter.

**Fix:**
1. Remove filters: `?limit=10` (get all runs)
2. Check filter values are correct
3. Try with no parameters first
4. See [API Reference](./API-REFERENCE.md) for parameter syntax

### Missing field in response

**Cause:** Schema or API version mismatch.

**Fix:**
1. Check `schema_version` field in the response
2. Refer to [Data Schema](./SCHEMA.md) for expected fields
3. Ensure you're parsing the JSON correctly

### "Attestation not found"

**Cause:** You're searching for an invalid UID on easscan.

**Fix:**
1. Copy the `attestation_uid` directly from the API response
2. Go to [base.easscan.org](https://base.easscan.org)
3. Paste the UID in the search box (not in URL bar)
4. Check the schema UID matches (benchmark vs. run log)

---

## Network Issues

### Connection timeout

**Cause:** Server not responding.

**Fix:**
1. Check your internet connection
2. Verify endpoint URL is correct
3. Try the health endpoint: `curl https://us-central1-fourmonth-73efe.cloudfunctions.net/api/health`
4. If health endpoint works but other endpoints don't, file an issue

### DNS resolution failed

**Cause:** Can't resolve `assaychain.com` (vanity host).

**Fix:**
1. Fall back to the canonical Cloud Functions URL: `https://us-central1-fourmonth-73efe.cloudfunctions.net/api`
2. Try a different DNS server (e.g. 1.1.1.1, 8.8.8.8)
3. Flush your local DNS cache

### SSL/TLS certificate error

**Cause:** HTTPS certificate invalid or expired.

**Fix:**
1. Check your system date/time
2. Update your certificate authorities
3. Try `curl --insecure` (for testing only, not production)

---

## Client-Side Issues

### "Module not found: x402-fetch"

**Fix:**
```bash
npm install x402-fetch
```

See [coinbase/x402 on GitHub](https://github.com/coinbase/x402) for the canonical clients (`x402-fetch`, `x402-axios`).

### JavaScript "fetch is not defined"

**Cause:** Using Node.js <17.5 or browser without fetch.

**Fix:**
1. Update Node.js to ≥18
2. Or import: `const fetch = require('node-fetch');`
3. Or use axios: `npm install axios`

### Python "ModuleNotFoundError: requests"

**Fix:**
```bash
pip install requests
```

### "CORS error" (browser)

**Cause:** Browser blocking cross-origin request.

**Fix:**
1. Use a CORS proxy (not recommended)
2. Make the request from backend/server instead
3. Server-side requests have no CORS limitations

---

## Verification Issues

### Can't find attestation on easscan

**Fix:**
1. Go to [base.easscan.org](https://base.easscan.org)
2. Paste the UID (from the API response) in the search box
3. Make sure you're on Base mainnet (not testnet)
4. Try searching for the schema UID instead

### IPFS gateway timeout

**Cause:** IPFS gateway slow or unavailable.

**Fix:**
1. Try a different gateway:
   - `https://cloudflare-ipfs.com/ipfs/Qm...`
   - `https://dweb.link/ipfs/Qm...`
   - `https://nft.storage/ipfs/Qm...`
2. Wait and retry
3. Use a local IPFS node if available

### Can't decode CID

**Cause:** CID format mismatch (CIDv0 vs CIDv1).

**Fix:**
1. CIDv0 starts with `Qm` (46 chars). CIDv1 typically starts with `bafy`/`bafk` (base32).
2. Either format works at any modern gateway:
   - `https://gateway.pinata.cloud/ipfs/<cid>`
   - `https://cloudflare-ipfs.com/ipfs/<cid>`
   - `https://dweb.link/ipfs/<cid>`
3. If a gateway 404s, try another — the IPFS data is the same.

---

## Rate Limiting

### "Too many requests"

**Cause:** You're calling too many endpoints in parallel.

**Current policy:** No hard rate limit (pay per call).

**Recommended:** Space requests 1–2 seconds apart.

---

## MCP Server Issues

### "Connection refused" (Claude Desktop)

**Fix:**
1. Verify the MCP URL is correct: `https://us-central1-fourmonth-73efe.cloudfunctions.net/api/mcp`
2. Test manually: `curl https://us-central1-fourmonth-73efe.cloudfunctions.net/api/mcp` (should error with JSON-RPC error, not connection refused)
3. Restart Claude Desktop
4. Check config file syntax (JSON must be valid)

### Tools not showing up

**Fix:**
1. Restart Claude Desktop — `mcp-remote` is launched on app start
2. Verify the MCP endpoint responds:
   ```bash
   curl -X POST https://us-central1-fourmonth-73efe.cloudfunctions.net/api/mcp \
     -H "Content-Type: application/json" \
     -H "Accept: application/json, text/event-stream" \
     -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
   ```
3. Confirm your `claude_desktop_config.json` uses `"command": "npx"` with `"args": ["-y", "mcp-remote", "<url>"]`
4. Validate the JSON syntax (a single trailing comma will silently disable the entry)

---

## Performance Issues

### Slow response time

**Typical latency:** 200–500ms (mostly network).

**If >2 seconds:**
1. Try again (might be transient)
2. Check your network speed
3. Try from a different location
4. File an issue with latency measurements

### High memory usage

**Fix:**
1. Limit query results: `?limit=10`
2. Don't fetch all runs at once
3. Paginate with `?offset=` and `?limit=`

---

## Still Stuck?

1. Check [FAQ](./FAQ.md) for common questions
2. Review [Code Examples](./EXAMPLES.md) for working samples
3. File an issue on GitHub with:
   - Exact error message
   - Endpoint you called
   - Request headers
   - Response body
   - Timestamp
4. Mention OS, language, library versions

---

## Common Mistakes

1. **Forgetting the `/api` prefix**
   - Wrong: `https://assaychain.com/benchmark/copper`
   - Right: `https://us-central1-fourmonth-73efe.cloudfunctions.net/api/benchmark/copper`

2. **Wrong commodity name**
   - Wrong: `cuprum`, `cu`, `copper_ore`
   - Right: `copper`

3. **Not waiting for payment confirmation**
   - Don't retry immediately after payment
   - Wait 5–10 seconds for Base confirmation

4. **Parsing JSON response as string**
   - Wrong: `console.log(response)`
   - Right: `const data = await response.json(); console.log(data)`

5. **Using testnet instead of mainnet**
   - Payments must be on Base **mainnet**, not testnet

---

## See Also

- [FAQ](./FAQ.md) — Frequently asked questions
- [API Reference](./API-REFERENCE.md) — Endpoint docs
- [x402 Guide](./x402-GUIDE.md) — Payment help
- [Getting Started](./GETTING-STARTED.md) — First-time walkthrough
