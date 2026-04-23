import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# MCP Deployment Guide

How to deploy LiteLLM as a central gateway for LLMs, MCP servers, and agents — including network topology, multi-instance setups, and the security tradeoffs of each.

---

## The core idea

LiteLLM acts as a single control plane for three resource types:

| Resource | What it is | Registered as |
|----------|-----------|---------------|
| **LLM** | Any model (OpenAI, Bedrock, Azure, Ollama, …) | `model_list` in config or via API |
| **MCP Server** | A tool server (filesystem, database, web search, …) | `mcp_servers` in config or via UI |
| **Agent** | An A2A-compatible agent endpoint | A2A routes |

All three are authenticated the same way (LiteLLM API key), rate-limited the same way, and appear in the same usage dashboard. You get a **central catalog** of every resource your org uses — without running separate registries for each type.

---

## Deployment topologies

### Option A: Single gateway (simplest)

One LiteLLM instance handles LLM routing, MCP tool calls, and A2A agent invocations.

```
Agents / AI clients
        │
        ▼
┌───────────────────────────────────┐
│         LiteLLM Gateway           │
│  /v1/chat/completions  (LLMs)     │
│  /mcp                  (tools)    │
│  /a2a                  (agents)   │
└───────┬───────┬──────────┬────────┘
        │       │          │
   OpenAI   MCP servers  Downstream
   Bedrock  (internal)    agents
   Azure    (public)
```

**When to use:**
- Internal deployments with a private network perimeter
- Teams where LLMs and MCP servers share the same trust boundary
- Easiest to operate — one service, one config, one set of API keys

**Tradeoff:** LLM traffic and MCP traffic share the same open port. If you open that port to the internet (for Claude Desktop or ChatGPT integration), your LLM endpoints are also reachable externally. Mitigate with the [public internet filter](./mcp_public_internet.md) and key-based auth.

---

### Option B: Separate LLM gateway and MCP gateway (recommended for most enterprises)

Split into two LiteLLM deployments: one for LLM routing (no internet exposure), one for MCP serving (optionally internet-facing).

```
Internal AI clients             External AI clients
        │                       (ChatGPT, Claude Desktop)
        │                               │
        ▼                               ▼
┌────────────────────┐     ┌────────────────────────┐
│  LLM Gateway       │     │  MCP Gateway           │
│  (no public port)  │     │  (port 443 / public)   │
│  /v1/chat/...      │     │  /mcp                  │
└────────┬───────────┘     └──────────┬─────────────┘
         │                            │
    LLM providers              MCP servers
    (OpenAI, Bedrock, …)       (internal + public)
```

**Why this separation matters:**

- The LLM gateway never needs to be internet-accessible. Credentials for OpenAI, Bedrock, etc. stay behind the firewall.
- The MCP gateway can be exposed on a public port. You can use [available_on_public_internet](./mcp_public_internet.md) per-server to control exactly which tools external callers see.
- A compromise of the MCP gateway does **not** expose LLM API keys.

**Config for LLM gateway (no MCP config needed):**

```yaml title="llm-gateway/config.yaml" showLineNumbers
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

  - model_name: claude-3-7-sonnet
    litellm_params:
      model: anthropic/claude-3-7-sonnet-20250219
      api_key: os.environ/ANTHROPIC_API_KEY
```

**Config for MCP gateway:**

```yaml title="mcp-gateway/config.yaml" showLineNumbers
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  store_model_in_db: true
  # Define which IP ranges count as "internal"
  mcp_internal_ip_ranges:
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    - "192.168.0.0/16"
    - "100.64.0.0/10"   # add your VPN/Tailscale range

mcp_servers:
  - server_name: filesystem
    url: http://mcp-filesystem.internal:8000/mcp
    transport: http
    available_on_public_internet: false  # internal only

  - server_name: web-search
    url: https://mcp.exa.ai/mcp
    transport: http
    available_on_public_internet: true   # visible to ChatGPT / Claude Desktop

# No model_list here — LLM routing is handled by the LLM gateway
```

---

### Option C: Multiple LiteLLM instances per team (for large orgs)

Some organizations run separate LiteLLM instances per team or environment (dev, staging, prod) and want a **central catalog** that aggregates all of them.

```
┌──────────────────────────────────────┐
│         Central LiteLLM              │
│  (catalog, unified auth, billing)    │
│  model registry + MCP registry       │
└──────┬──────────────┬────────────────┘
       │              │
       ▼              ▼
┌─────────────┐  ┌──────────────┐
│  Team A     │  │  Team B      │
│  LiteLLM    │  │  LiteLLM     │
│  (Bedrock)  │  │  (Azure OAI) │
└─────────────┘  └──────────────┘
```

**How to implement:**

1. Each team runs their own LiteLLM instance with their own model list.
2. The central instance registers team instances as **upstream models** (using LiteLLM's [model endpoint routing](./proxy/configs.md)):

```yaml title="central/config.yaml" showLineNumbers
model_list:
  # Route to Team A's instance
  - model_name: team-a/gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_base: http://litellm-team-a.internal:4000
      api_key: os.environ/TEAM_A_KEY

  # Route to Team B's instance
  - model_name: team-b/azure-gpt4
    litellm_params:
      model: openai/gpt-4
      api_base: http://litellm-team-b.internal:4000
      api_key: os.environ/TEAM_B_KEY

  # MCP servers still registered centrally
mcp_servers:
  - server_name: shared-db
    url: http://db-mcp.internal:8000/mcp
    transport: http
```

3. Teams authenticate to the central instance with scoped API keys. Access to specific models and MCP servers is controlled with [key permissions](./proxy/virtual_keys.md) and [team-based access](./proxy/team_based_routing.md).

---

## Central catalog of LLMs, MCPs, and agents

LiteLLM provides a single place to discover every resource in your org:

| Endpoint | What it returns |
|----------|----------------|
| `GET /v1/models` | All LLMs registered in this instance |
| `GET /mcp/servers` (UI) or `GET /v1/mcp/server` | All MCP servers |
| `GET /mcp` | MCP tool listing (all tools from all servers) |
| `GET /.well-known/agent.json` (A2A) | Agent card for this instance |

**MCP registry** (opt-in): Enable `enable_mcp_registry: true` in `general_settings` and LiteLLM exposes a `GET /v1/mcp/registry.json` endpoint that any MCP client can use to discover all registered servers. Useful for Claude Desktop or Cursor to auto-populate the server list.

```yaml title="config.yaml"
general_settings:
  enable_mcp_registry: true
```

Then in Claude Desktop:
```json
{
  "mcpServers": {
    "litellm": {
      "url": "https://your-litellm.example.com/mcp",
      "headers": { "Authorization": "Bearer sk-..." }
    }
  }
}
```

---

## Security considerations

### The open-port problem

This is the most common mistake. If your LiteLLM instance runs on a single port and you expose that port to the internet (so Claude Desktop can connect), your `/v1/chat/completions` endpoint is also reachable from the internet — and so are all your LLM API keys.

**Mitigations, in order of preference:**

1. **Separate deployments** (Option B above) — the LLM gateway never gets a public port. This is the cleanest solution.
2. **Network-level firewall** — if you must run a single instance, use a firewall/WAF to block `/v1/chat/completions` from public IPs while allowing `/mcp`.
3. **Require auth on every endpoint** — LiteLLM requires a Bearer token on all endpoints when `master_key` is set. Combined with short-lived keys, this limits blast radius.

### MCP servers can reach the public internet

An MCP server is just an HTTP server. If you register `https://mcp.exa.ai/mcp`, LiteLLM will make outbound requests to the public internet when tools are called. In a locked-down network (no outbound access), those calls will silently fail.

**What to check before adding an MCP server:**
- Does your network allow outbound HTTPS to that host?
- Does your security policy allow data leaving the network to that third party?
- Does the MCP server require credentials (API key, OAuth)? Rotate them if they get included in LiteLLM config.

**For air-gapped or restricted networks:** Only register MCP servers that run inside your perimeter. Use the `available_on_public_internet: false` setting (the default) so those servers never appear to external callers even if you open a port later.

### "If it's on the same instance, everything is open for everyone"

This is true by default — all authenticated callers can list and call all MCP tools unless you add explicit access controls. LiteLLM has several layers:

| Control | Where to configure |
|---------|------------------|
| Which tools a key can call | [Key-level MCP permissions](./mcp_control.md) |
| Which tools a team can call | [Team-level MCP permissions](./mcp_control.md) |
| Block external callers from seeing internal servers | [available_on_public_internet](./mcp_public_internet.md) |
| Verify requests actually came through LiteLLM | [MCP JWT Signer (Zero Trust)](./mcp_zero_trust.md) |
| Block sensitive data in tool responses | [MCP Guardrails](./mcp_guardrail.md) |

A minimal secure config for an enterprise deployment:

```yaml title="config.yaml" showLineNumbers
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  store_model_in_db: true
  mcp_internal_ip_ranges:
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    - "192.168.0.0/16"

mcp_servers:
  - server_name: internal-db
    url: http://db-mcp.internal:8000/mcp
    transport: http
    available_on_public_internet: false  # never visible to external callers

guardrails:
  - guardrail_name: mcp-jwt-signer
    litellm_params:
      guardrail: mcp_jwt_signer
      mode: pre_mcp_call
      default_on: true
      issuer: "https://your-litellm.example.com"
```

And issue scoped API keys to each team/agent rather than handing out the master key:

```bash
# Create a key for Team A that can only access their MCP servers and LLMs
curl -X POST https://your-litellm.example.com/key/generate \
  -H "Authorization: Bearer $MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "team_id": "team-a",
    "models": ["team-a/gpt-4o"],
    "mcp_servers": ["internal-db"],
    "duration": "30d"
  }'
```

---

## Decision guide

| Situation | Recommended approach |
|-----------|---------------------|
| Internal-only, all tools trusted | Single gateway, no public port |
| Need Claude Desktop / ChatGPT integration | Separate MCP gateway on public port, LLM gateway stays internal |
| Multiple teams, different LLM providers | Option C — central catalog with per-team instances upstream |
| Air-gapped network | Single gateway, register only internal MCP servers |
| External MCP servers in a restricted network | Check outbound policy first; use `available_on_public_internet: false` |
| Need to prove requests came through LiteLLM | [MCP JWT Signer](./mcp_zero_trust.md) on all tool calls |

---

## Related

- [MCP Overview](./mcp.md) — getting started with MCP in LiteLLM
- [Public Internet Filter](./mcp_public_internet.md) — control which MCP servers are visible externally
- [MCP Access Control](./mcp_control.md) — key and team-level tool permissions
- [MCP Zero Trust](./mcp_zero_trust.md) — JWT signing to verify requests came through LiteLLM
- [MCP Guardrails](./mcp_guardrail.md) — block sensitive data in tool responses
- [Proxy Deployment](./proxy/deploy.md) — general proxy deployment options
