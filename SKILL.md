---
name: arona-gateway
description: Pay-per-call AI inference with no signup, no card, no KYC. Pay 0.01 USDC per call over x402 (Base mainnet) or 2 sats over Lightning (L402). OpenAI-compatible endpoint with automatic provider failover.
metadata:
  {
    "openclaw": {
      "emoji": "🧠",
      "requires": {},
      "network": ["https://fancy-stallion-9156.aronaplana-os.deno.net"]
    }
  }
---

# Arona Gateway — pay-per-call AI inference

An OpenAI-compatible inference endpoint for agents that **do not have an account, a card, or a KYC-checked identity**. Two payment rails:

| Rail | Asset | Price | Who it's for |
|---|---|---|---|
| **x402** | USDC on Base (`eip155:8453`) | **0.01 USDC / call** | agents with an EVM wallet |
| **L402** | Lightning (sats) | **2 sats / call** | agents with a Lightning wallet |
| Free tier | — | free | `Bearer arona`, shared capacity |

Base URL: `https://fancy-stallion-9156.aronaplana-os.deno.net`

## How to call it (x402)

```bash
# 1) Ask for the payment challenge (HTTP 402)
curl -s https://fancy-stallion-9156.aronaplana-os.deno.net/x402/quote

# Response contains:
#   accepts[0] = {scheme:"exact", network:"eip155:8453", asset:"0x8335...2913",
#                 amount:"10000", payTo:"0x265A2CD6Bf17fE20FcE68Db5473EA29a1Ac389d1"}
#   extensions.bazaar = machine-readable discovery metadata
#
# 2) Sign an EIP-3009 transferWithAuthorization for that amount, then retry with
#    the base64-encoded payment payload in the PAYMENT-SIGNATURE header:
curl -s -X POST https://fancy-stallion-9156.aronaplana-os.deno.net/x402/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "PAYMENT-SIGNATURE: <base64 payment payload>" \
  -d '{"model":"auto","messages":[{"role":"user","content":"hello"}]}'
```

The server verifies through the **PayAI facilitator** (`https://facilitator.payai.network`, no API key required) and settles on-chain. The response includes `_x402.transaction` with the settlement tx hash.

## How to call it (L402 / Lightning)

```bash
# 1) Get a quote (HTTP 402 + BOLT11 invoice)
curl -s https://fancy-stallion-9156.aronaplana-os.deno.net/paid/quote

# 2) Pay the invoice with any Lightning wallet, then poll for the token:
curl -s "https://fancy-stallion-9156.aronaplana-os.deno.net/paid/check?receipt=<receipt>"
#    -> { "settled": true, "token": "<hmac token>", "quota": 20, "ttl_sec": 3600 }

# 3) Use the token for up to 20 calls / 1 hour:
curl -s -X POST https://fancy-stallion-9156.aronaplana-os.deno.net/paid/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: L402 <token>" \
  -d '{"model":"auto","messages":[{"role":"user","content":"hello"}]}'
```

## Models

- `auto` — automatic failover: Cloudflare Workers AI (llama-3.3-70b / gpt-oss-120b) → pollinations
- `@cf/meta/llama-3.3-70b-instruct-fp8-fast` — fast general model
- `@cf/openai/gpt-oss-120b` — 128k context, reasoning
- `@cf/qwen/qwen2.5-coder-32b-instruct` — code
- `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` — reasoning
- `pollinations:openai-fast`, `llm7:gpt-oss:20b` — keyless upstreams

`GET /v1/models` lists the current set (free tier requires `Authorization: Bearer arona`).

## Notes for agents

- The 402 body is spec-shaped x402 v2 (`x402Version: 2`, `accepts[]`, `resource`, `extensions.bazaar`).
- Settlement is idempotent per payment payload; tokens for L402 are stateless HMAC (quota + expiry inside).
- Prices: **0.01 USDC** (x402) / **2 sats** (L402). The free tier stays free; the paid rails exist for agents that cannot sign up.
- Source: https://github.com/aronaplana-os/arona-cloud-gateway
