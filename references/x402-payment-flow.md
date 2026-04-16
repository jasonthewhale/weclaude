# x402 Payment Flow — Header Assembly & Replay

This reference covers the technical detail for building the `PAYMENT-SIGNATURE` / `X-PAYMENT` header and replaying the request to `/v1/buyer/topup` after signing.

## Header Name

Determine from `decoded.x402Version` (read from the 402 challenge JSON):

| x402Version | Header name |
|---|---|
| `>= 2` | `PAYMENT-SIGNATURE` |
| `< 2` or absent | `X-PAYMENT` |

## Building the Payment Payload

After signing with `onchainos payment x402-pay`, you have:
- `SIGNATURE` — the hex signature string (from `data.signature` or top-level `signature`)
- `AUTHORIZATION` — the authorization object (from `data.authorization` or top-level `authorization`)
- `DECODED` — the full 402 challenge JSON (the 402 response body, saved to `/tmp/weclaude-challenge.json`)
- `OPTION` — `DECODED.accepted` (the payment option object from the body)

Construct the payload:

```json
{
  "x402Version": <decoded.x402Version>,
  "resource": "<decoded.resource>",
  "accepted": <option>,
  "payload": {
    "signature": "<signature>",
    "authorization": <authorization>
  }
}
```

Base64-encode the JSON (no line breaks):

```bash
python3 -c "
import json, base64, sys

with open('/tmp/weclaude-challenge.json') as f:
    decoded = json.load(f)

option = decoded['accepted']
signature = sys.argv[1]
authorization = json.loads(sys.argv[2])

payload = {
    'x402Version': decoded.get('x402Version', 1),
    'resource': decoded['resource'],
    'accepted': option,
    'payload': {'signature': signature, 'authorization': authorization}
}
print(base64.b64encode(json.dumps(payload, separators=(',',':')).encode()).decode())
" "$SIGNATURE" "$AUTHORIZATION_JSON" > /tmp/weclaude-payment-header.txt

HEADER_VALUE=$(cat /tmp/weclaude-payment-header.txt)
```

## Replaying the Request

**Both headers are required.** `Content-Type: application/json` activates the Express body parser — without it the request may be rejected or the payment middleware may not process correctly.

```bash
curl -s -X POST "https://api.weclaude.cc/v1/buyer/topup" \
  -H "Content-Type: application/json" \
  -H "<HEADER_NAME>: $HEADER_VALUE"
```

## Success Response

```json
{
  "api_key": "sk-x402-...",
  "balance": "$0.10",
  "pricing": "real token usage — varies by model",
  "withdraw_url": "/v1/buyer/withdraw",
  "usage": "Authorization: Bearer sk-x402-..."
}
```

## Error Responses

The server returns **402 for ALL payment failure cases** — missing header, malformed header, invalid signature, and wrong amount all produce the same 402 with a fresh challenge. Do not try to distinguish failure modes from the status code.

| HTTP | Meaning | What to check |
|---|---|---|
| `402` | Payment missing, malformed, or failed verification | 1. Is `PAYMENT-SIGNATURE` header present? 2. Is the value valid base64 of a JSON object? 3. Does the payload have the correct structure (`x402Version`, `resource`, `accepted`, `payload`)? 4. Is `Content-Type: application/json` set? |
| `200` | Payment accepted | Success — response contains `api_key` |
| `502` | Facilitator error | OKX x402 service issue — retry after a few seconds |

## Extracting Signature and Authorization from onchainos

The `onchainos payment x402-pay` response may nest fields under `data`:

```json
{
  "data": {
    "signature": "0x...",
    "authorization": { "from": "0x...", "to": "0x...", ... }
  }
}
```

Or return them at the top level:

```json
{
  "signature": "0x...",
  "authorization": { "from": "0x...", "to": "0x...", ... }
}
```

Always check both. Use `jq` or `python3 -c "import json,sys; d=json.load(sys.stdin); ..."` to extract reliably.

## Scheme

The server accepts both `exact` and `aggr_deferred` schemes. The 402 response's `accepted` field specifies which scheme to use — always use the `accepted` object as-is from the 402 body (it defaults to `exact`). Do not switch schemes when debugging payment failures.

## Network Reference

| Field | Value |
|---|---|
| Network | X Layer |
| Chain ID | `eip155:196` |
| Token | USDG |
| Decimals | 6 |
| Topup amount | `100000` atomic units = `0.10` USDG |
