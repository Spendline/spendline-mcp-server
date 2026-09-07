# Spendline MCP Server

**Spendline is the financial control layer for AI spend.** It sits in the request
path between your application and its model providers, attributes every call to a
customer, agent, workflow and cost centre, and enforces budgets *before* the money
is spent. A call that would breach a cap is refused with HTTP 402 and never reaches
the provider.

This repository documents Spendline's **hosted, remote MCP server**. There is
nothing to install and no package to run: the server is already live.

```
https://www.spendline.ai/mcp
```

Transport: Streamable HTTP. Protocol version: 2025-06-18.

---

## What it is for

| You want to | Spendline |
|---|---|
| Stop an agent spending more than $500 a month | Yes, hard cap, refused before the call |
| Know AI cost and gross margin per customer | Yes, per-request attribution into a ledger |
| One budget spanning OpenAI **and** Anthropic | Yes, enforcement is in the request path |
| Reconcile AI spend against the provider invoice | Yes, month close with immutable closed periods |
| Trace prompts, run evals, debug latency | **No.** Use Langfuse, LangSmith or Helicone |

Spendline is not an observability tool and does not try to be. It sits alongside one.

## Ten providers

OpenAI, Anthropic, Google Gemini, xAI, Mistral, DeepSeek, Alibaba Qwen,
Together AI, Fireworks AI and Groq.

## Connect the MCP server

Any MCP client that accepts a remote streamable-HTTP endpoint:

```json
{
  "mcpServers": {
    "spendline": {
      "url": "https://www.spendline.ai/mcp"
    }
  }
}
```

Four tools answer **with no credential at all**, so an agent can evaluate
Spendline before anyone signs up:

- `spendline_when_to_use`: intent to capability map, plus the cases where a
  *different* tool is the right answer, and honest comparisons against LiteLLM,
  Portkey, Cloudflare AI Gateway and the observability tools.
- `spendline_list_providers`: every provider, the accepted request shapes, and
  the exact base URL per SDK.
- `spendline_get_integration_instructions`: the full integration documents.
- `spendline_get_onboarding_instructions`: how to get an account and a key.

The remaining tools are tenant-scoped and need a Spendline API key in the
`x-spendline-key` header: read spend, list budgets, list policies, list budget
scopes, check integration status, and create a budget. Note the only mutating
tool **adds** a restriction. There is deliberately no agent-reachable path to
billing, provider-key storage, budget raises or deletes, or month close.

## Integrate the proxy (this is what actually enforces)

Connecting the MCP server lets an agent ask and manage. Enforcement starts when
your provider traffic is routed through Spendline, which is a base URL change.

**OpenAI SDK, base URL WITH `/v1`:**

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://www.spendline.ai/v1",
  apiKey: process.env.OPENAI_API_KEY,            // your own provider key
  defaultHeaders: {
    "x-spendline-key": process.env.SPENDLINE_API_KEY,
    "x-customer-id": "acme-inc",
    "x-agent-id": "support-bot",
    "x-spendline-tags": JSON.stringify({ team: "support", env: "prod" }),
  },
});
```

**Anthropic SDK, base URL WITHOUT `/v1`:**

```javascript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  baseURL: "https://www.spendline.ai",
  apiKey: process.env.ANTHROPIC_API_KEY,         // your own provider key
  defaultHeaders: {
    "x-spendline-key": process.env.SPENDLINE_API_KEY,
    "x-customer-id": "acme-inc",
    "x-agent-id": "support-bot",
    "x-spendline-tags": JSON.stringify({ team: "support" }),
  },
});
```

The asymmetry is real: the OpenAI SDK appends `/chat/completions` to your base
URL, the Anthropic SDK appends `/v1/messages`. Do not "fix" it.

**Never send the Spendline key as `x-api-key`.** Anthropic uses `x-api-key` for
its own credential.

## Response codes worth knowing

| Code | Meaning | What an agent should do |
|---|---|---|
| 402 | A budget blocked this call before any spend | Correct, intended behaviour. Surface it. **Never** retry against the provider directly, that defeats the product. |
| 403 | A policy or governance rule blocked the call | Surface it |
| 400 | Attribution missing | Add `x-agent-id`, `x-customer-id` and `x-spendline-tags` with a cost-centre key |

## Machine-readable surfaces

Everything below is public, unauthenticated, and generated from a single manifest,
so it cannot drift from what the proxy actually does:

- https://www.spendline.ai/llms.txt
- https://www.spendline.ai/llms-full.txt
- https://www.spendline.ai/openapi.json
- https://www.spendline.ai/.well-known/mcp.json
- https://www.spendline.ai/agents/capabilities.json
- https://www.spendline.ai/agents/when-to-use.json
- https://www.spendline.ai/for-agents/

## Links

- Website: https://www.spendline.ai
- Guides: https://www.spendline.ai/guides/
- Sign up: https://www.spendline.ai/signup

## License

MIT for this documentation repository. The Spendline service itself is a
commercial hosted product.
