# x402 Payment Flow — Header Assembly & Replay

This reference covers the technical detail for building the `PAYMENT-SIGNATURE` / `X-PAYMENT` header and replaying the request to `/v1/topup` after signing.

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
- `DECODED` — the full 402 challenge JSON (parsed from the `PAYMENT-REQUIRED` header)
- `OPTION` — `DECODED.accepts[0]`

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

option = decoded['accepts'][0]
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

```bash
curl -s -X POST "<SERVER_URL>/v1/topup" \
  -H "Content-Type: application/json" \
  -H "<HEADER_NAME>: $HEADER_VALUE"
```

## Success Response

```json
{
  "api_key": "sk-x402-...",
  "balance": "$0.10",
  "pricing": "real token usage — varies by model",
  "close_url": "/v1/close",
  "usage": "Authorization: Bearer sk-x402-..."
}
```

## Error Responses

| HTTP | Body | Meaning |
|---|---|---|
| `400` | `{ "error": "..." }` | Malformed payment header — check JSON structure and base64 encoding |
| `402` | Challenge JSON | Payment not yet accepted — header was missing or invalid |
| `403` | `{ "error": "payment verification failed" }` | Signature invalid or wrong asset/amount |
| `500` | Server error | Facilitator issue — retry or check server logs |

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

## Network Reference

| Field | Value |
|---|---|
| Network | X Layer |
| Chain ID | `eip155:196` |
| Token | USDG |
| Decimals | 6 |
| Topup amount | `100000` atomic units = `0.10` USDG |
