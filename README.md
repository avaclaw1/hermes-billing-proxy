# Hermes Routing Layer

Route [Hermes Agent](https://github.com/NousResearch/hermes-agent) API requests through your Claude Max/Pro subscription instead of Extra Usage billing.

Forked from [zacdcook/openclaw-billing-proxy](https://github.com/zacdcook/openclaw-billing-proxy) and adapted for Hermes Agent's architecture.

## How It Works

The proxy sits between Hermes and Anthropic's API, performing multi-layer bidirectional request/response processing:

**Outbound (request to API):**
1. **Billing Header** — Injects Claude Code billing identifier with dynamic SHA256 fingerprint
2. **String Sanitization** — Replaces Hermes-specific trigger phrases (Hermes Agent → Claude Code, Nous Research → Anthropic, etc.)
3. **Tool Name Bypass** — Renames all 49 Hermes tool names (including `mcp_`-prefixed) to PascalCase Claude Code convention
4. **System Prompt Processing** — Paraphrases Hermes boilerplate sections while preserving user content (SOUL.md, MEMORY, USER PROFILE, skills catalog)
5. **Unused Skills Removal** — Strips configurable list of unused skills from `<available_skills>` XML
6. **Property Renaming** — Renames Hermes-specific schema properties
7. **CC Tool Stubs** — Injects Claude Code tool stubs to match expected tool set
8. **Stainless SDK Headers** — Full Claude Code identity headers
9. **Thinking Block Protection** — Masks `thinking`/`redacted_thinking` blocks before transforms to preserve byte-equality

**Inbound (response to Hermes):**
10. **Full Reverse Mapping** — Restores all tool names, property names, and identifiers in both SSE streaming and JSON responses
11. **Event-Aware SSE** — Processes per SSE event with thinking block passthrough

## Key Differences from OpenClaw Proxy

| Feature | OpenClaw Proxy | Hermes Proxy |
|---|---|---|
| Tool prefix | None | `mcp_` (Hermes prefixes all tools with `mcp_` when using OAuth) |
| System prompt | Clean split (config vs user docs) | Mixed in one block (requires surgical regex) |
| Boilerplate handling | Strip entire config section | Paraphrase 3 sections, keep user content |
| Tool descriptions | Stripped (Layer 5) | **Kept** — model needs them for functionality |
| Dynamic MCP tools | N/A | Regex rename `mcp_*` → `Mcp*` (PascalCase) |
| Token refresh | Claude CLI based | Claude CLI based with exponential backoff |

## Requirements

- Node.js 18+
- Claude Max or Pro subscription
- Claude Code CLI installed and authenticated (`claude auth login`)
- Hermes Agent installed

## Setup

```bash
# 1. Start the proxy
node proxy.js

# 2. Update Hermes config (~/.hermes/config.yaml)
model:
  default: claude-opus-4-6
  provider: anthropic
  base_url: http://127.0.0.1:18802
  api_key: dummy
providers:
  anthropic:
    base_url: http://127.0.0.1:18802
    api_key: dummy

# 3. Restart Hermes gateway
```

## Systemd Service (Linux)

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/hermes-proxy.service << EOF
[Unit]
Description=Hermes Billing Proxy
After=network.target

[Service]
ExecStart=/usr/bin/node /path/to/proxy.js
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable hermes-proxy.service
systemctl --user start hermes-proxy.service
```

## Configuration

Create a `config.json` alongside `proxy.js` to override defaults:

```json
{
  "port": 18802,
  "refreshEnabled": true,
  "refreshThresholdMinutes": 2,
  "refreshRetrySeconds": 15
}
```

### Removing Unused Skills

The proxy removes skills that aren't needed from the `<available_skills>` catalog to reduce body size. Edit the `REMOVE_SKILLS` set in `proxy.js` to customize which skills are removed.

## Health Check

```bash
curl http://127.0.0.1:18802/health
```

## Token Refresh

The proxy automatically refreshes the Claude Code OAuth token when it's within 2 minutes of expiry. It:

1. Triggers `claude -p "ping"` to refresh the credential store
2. Re-reads the refreshed token from `~/.claude/.credentials.json`
3. Retries with exponential backoff (15s → 30s → 60s... capped at 10m)
4. Gives up after 20 consecutive failures

No cron job needed.

## How Detection Works

Anthropic uses multiple layers to detect non-Claude-Code clients:

1. **Billing header** — Checks for `x-anthropic-billing-header` in the system prompt
2. **String triggers** — Scans for known agent-specific phrases
3. **Tool name fingerprint** — Identifies the agent by its combination of tool names
4. **System prompt template** — Matches known boilerplate text blocks (exact/fuzzy matching against published source code)
5. **Body scoring** — Composite score across all signal sources

The proxy addresses all five layers. The key insight for Hermes: detection matches against **exact boilerplate text blocks** from Hermes's source code, not individual keywords. Short paraphrases with the same tool names pass because they don't match the known templates.

## License

MIT
