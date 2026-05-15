# Code Examples

Ready-to-use examples in cURL, JavaScript, and Python.

---

## Test the Free Endpoint

All languages — no API key needed.

### cURL

```bash
curl https://assaychain.com/api/health
```

### JavaScript (Node.js)

```javascript
const response = await fetch('https://assaychain.com/api/health');
const data = await response.json();
console.log(data);
```

### Python

```python
import requests

response = requests.get('https://assaychain.com/api/health')
data = response.json()
print(data)
```

---

## Call a Gated Endpoint (No Payment)

This will return `402 Payment Required`.

### cURL

```bash
curl -i https://assaychain.com/api/benchmark/copper
```

### 402 envelope (no payment)

```
HTTP/2 402

{
  "error": "Payment required",
  "accepts": [{
    "scheme": "exact",
    "network": "base",
    "maxAmountRequired": "100000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b3566dA4ef5",
    "payTo": "0x750977976Ab85A4Ce5AAbb2e1a9fc80a633f2769",
    "resource": "https://assaychain.com/api/benchmark/copper",
    "maxTimeoutSeconds": 60
  }],
  "x402Version": 1
}
```

### JavaScript

```javascript
const response = await fetch('https://assaychain.com/api/benchmark/copper');
console.log(response.status); // 402

const data = await response.json();
console.log(data);
```

### Python

```python
import requests

response = requests.get('https://assaychain.com/api/benchmark/copper')
print(response.status_code)  # 402

data = response.json()
print(data)
```

---

## Call with x402 Payment (Claude Desktop)

Easiest method — use [`mcp-remote`](https://github.com/geelen/mcp-remote) to bridge Claude Desktop's stdio MCP transport to our Streamable HTTP endpoint.

### Setup

Add to your Claude Desktop config:

```json
{
  "mcpServers": {
    "mineral-intel-benchmark": {
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

### Usage in Chat

```
You: "Use ask_sales_agent to summarise what this API sells"
Claude: [calls ask_sales_agent — free]
```

Note: MCP cannot carry the x402 `X-PAYMENT` header. For paid tools, settle the payment with an x402 client against the REST endpoint directly.

---

## Query Run Data

### cURL (free — returns 402)

```bash
# Get first 10 runs
curl "https://assaychain.com/api/benchmark/ultrasound-grooved-tray?limit=10"

# Filter: placer slurry, recovery > 70%
curl "https://assaychain.com/api/benchmark/ultrasound-grooved-tray?feed_type=placer_slurry&min_recovery_pct=70"

# Only attested runs
curl "https://assaychain.com/api/benchmark/ultrasound-grooved-tray?attested_only=true&limit=5"
```

### JavaScript (`x402-fetch`)

```javascript
import { wrapFetchWithPayment } from 'x402-fetch';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount(process.env.WALLET_PRIVATE_KEY);
const fetchPaid = wrapFetchWithPayment(fetch, account);

// First call triggers 402; the wrapper auto-settles and retries.
const res = await fetchPaid(
  'https://assaychain.com/api/benchmark/ultrasound-grooved-tray?limit=5&attested_only=true'
);
const data = await res.json();
console.log(`Got ${data.runs.length} runs`);
```

### Python (manual payment simulation)

```python
import requests
import json

# Simulate querying with payment
# (In production, use x402 SDK for real payment signing)

headers = {
    'X-PAYMENT': '<your-valid-payment-signature>',
}

response = requests.get(
    'https://assaychain.com/api/benchmark/ultrasound-grooved-tray?limit=5',
    headers=headers
)

if response.status_code == 200:
    runs = response.json()['runs']
    for run in runs:
        print(f"Run {run['run_id']}: {run['recovery_pct_estimated']}% recovery")
elif response.status_code == 402:
    print("Payment required:", response.json())
```

---

## Parse a Benchmark Response

Responses are wrapped: `{ ok, data, provenance, meta }`.

### JavaScript

```javascript
const res = await fetchPaid('https://assaychain.com/api/benchmark/copper');
const body = await res.json();
const { data, provenance } = body;

console.log(`Commodity: ${data.commodity} (${data.symbol})`);
console.log(`Spot price: $${data.price.spot_usd_per_lb}/lb`);
console.log(`World mine output (kt): ${data.production.world_mine_output_2025_ktonnes}`);
console.log(`Flotation recovery (typical): ${data.recovery_benchmarks.flotation_pct.typical}%`);
console.log(`Verify at: ${provenance.verify_url}`);
```

### Python

```python
import requests

res = requests.get(
    'https://assaychain.com/api/benchmark/copper',
    headers={'X-PAYMENT': '<base64 x402 payload>'}
)
body = res.json()
data = body['data']

print(f"Commodity: {data['commodity']} ({data['symbol']})")
print(f"Spot price: ${data['price']['spot_usd_per_lb']}/lb")
print(f"World mine output (kt): {data['production']['world_mine_output_2025_ktonnes']}")
print(f"Verify at: {body['provenance']['verify_url']}")
```

---

## Verify Attestation on-chain

### JavaScript

```javascript
// Build attestation explorer URL
const body = await res.json();
const uid = body.provenance.attestation_uid;
const easscanUrl = `https://base.easscan.org/attestation/view/${uid}`;
console.log(`Verify at: ${easscanUrl}`);
```

### Python

```python
body = res.json()
uid = body['provenance']['attestation_uid']

easscan_url = f"https://base.easscan.org/attestation/view/{uid}"
print(f"Verify at: {easscan_url}")
```

---

## Fetch Data via IPFS

### cURL

```bash
# Via Cloudflare gateway
curl https://cloudflare-ipfs.com/ipfs/Qm...

# Via dweb.link
curl https://dweb.link/ipfs/Qm...

# Via local IPFS node (if running)
ipfs get Qm...
```

### JavaScript

```javascript
async function fetchFromIPFS(cid) {
  const gateways = [
    'https://cloudflare-ipfs.com/ipfs/',
    'https://dweb.link/ipfs/',
  ];

  for (const gateway of gateways) {
    try {
      const response = await fetch(gateway + cid);
      if (response.ok) {
        return await response.json();
      }
    } catch (e) {
      console.error(`Gateway ${gateway} failed:`, e);
    }
  }

  throw new Error('All IPFS gateways failed');
}

// Usage
const benchmark = await response.json();
const resultData = await fetchFromIPFS(benchmark.result_cid);
console.log(resultData);
```

### Python

```python
import requests

def fetch_from_ipfs(cid):
    gateways = [
        'https://cloudflare-ipfs.com/ipfs/',
        'https://dweb.link/ipfs/',
    ]

    for gateway in gateways:
        try:
            response = requests.get(gateway + cid)
            if response.status_code == 200:
                return response.json()
        except Exception as e:
            print(f"Gateway {gateway} failed: {e}")

    raise Exception("All IPFS gateways failed")

# Usage
benchmark = response.json()
result_data = fetch_from_ipfs(benchmark['result_cid'])
print(result_data)
```

---

## Build a Simple Agent

### JavaScript (Conceptual)

```javascript
class MineralIntelligenceAgent {
  constructor() {
    this.baseURL = 'https://assaychain.com/api';
  }

  async getBenchmark(commodity) {
    const response = await fetch(
      `${this.baseURL}/benchmark/${commodity}`,
      {
        headers: {
          'X-PAYMENT': await this.getPaymentSignature(),
        },
      }
    );

    if (response.status === 402) {
      throw new Error('Payment failed');
    }

    return response.json();
  }

  async getRuns(filters = {}) {
    const params = new URLSearchParams(filters);
    const response = await fetch(
      `${this.baseURL}/benchmark/ultrasound-grooved-tray?${params}`,
      {
        headers: {
          'X-PAYMENT': await this.getPaymentSignature(),
        },
      }
    );

    return response.json();
  }

  async getPaymentSignature() {
    // Implement x402 payment signing
    // See x402 spec for details
    return '<payment-signature>';
  }
}

// Usage
const agent = new MineralIntelligenceAgent();

const copper = await agent.getBenchmark('copper');
console.log(`Copper spot price: $${copper.data.price.spot_usd_per_lb}/lb`);

const runs = await agent.getRuns({
  feed_type: 'placer_slurry',
  min_recovery_pct: 70,
});
console.log(`Found ${runs.runs.length} high-recovery runs`);
```

### Python (Conceptual)

```python
class MineralIntelligenceAgent:
    def __init__(self):
        self.base_url = 'https://assaychain.com/api'

    def get_benchmark(self, commodity):
        response = requests.get(
            f'{self.base_url}/benchmark/{commodity}',
            headers={'X-PAYMENT': self.get_payment_signature()}
        )
        if response.status_code == 402:
            raise Exception('Payment failed')
        return response.json()

    def get_runs(self, **filters):
        params = '&'.join([f'{k}={v}' for k, v in filters.items()])
        response = requests.get(
            f'{self.base_url}/benchmark/ultrasound-grooved-tray?{params}',
            headers={'X-PAYMENT': self.get_payment_signature()}
        )
        return response.json()

    def get_payment_signature(self):
        # Implement x402 payment signing
        return '<payment-signature>'

# Usage
agent = MineralIntelligenceAgent()

copper = agent.get_benchmark('copper')
print(f"Copper spot price: ${copper['data']['price']['spot_usd_per_lb']}/lb")

runs = agent.get_runs(
    feed_type='placer_slurry',
    min_recovery_pct=70
)
print(f"Found {len(runs['runs'])} high-recovery runs")
```

---

## Error Handling

### JavaScript

```javascript
async function safeCall(endpoint) {
  try {
    const response = await fetch(endpoint);

    switch (response.status) {
      case 200:
        return await response.json();
      case 402:
        console.error('Payment required:', await response.json());
        break;
      case 404:
        console.error('Not found');
        break;
      case 500:
        console.error('Server error');
        break;
      default:
        console.error(`HTTP ${response.status}`);
    }
  } catch (error) {
    console.error('Network error:', error);
  }
}
```

### Python

```python
def safe_call(endpoint):
    try:
        response = requests.get(endpoint)

        if response.status_code == 200:
            return response.json()
        elif response.status_code == 402:
            print('Payment required:', response.json())
        elif response.status_code == 404:
            print('Not found')
        elif response.status_code == 500:
            print('Server error')
        else:
            print(f'HTTP {response.status_code}')
    except requests.RequestException as e:
        print(f'Network error: {e}')
```

---

## See Also

- [API Reference](./API-REFERENCE.md) — All endpoints
- [Getting Started](./GETTING-STARTED.md) — First-time guide
- [x402 Guide](./x402-GUIDE.md) — Payment details
- [Troubleshooting](./TROUBLESHOOTING.md) — Error handling
