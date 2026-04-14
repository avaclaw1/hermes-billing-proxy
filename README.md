# Hermes Billing Proxy

Route [Hermes](https://github.com/lbr77/hermes) API requests through your Claude Max/Pro subscription instead of Extra Usage billing.

Forked from [zacdcook/openclaw-billing-proxy](https://github.com/zacdcook/openclaw-billing-proxy) and adapted for Hermes's architecture.

## How It Works

The proxy sits between Hermes and Anthropic's API, performing multi-layer bidirectional request/response processing:

**Outbound (request to API):**
1. **Billing Header** — Injects Hermes Agent billing identifier with dynamic SHA256 fingerprint
2. **String Sanitization** — Replaces framework-specific trigger phrases with neutral equivalents
3. **Tool Name Bypass** — Renames all 49 Hermes tool names (including `mcp_`-prefixed) to PascalCase Hermes Agent convention
4. **System Prompt Processing** — Paraphrases boilerplate sections while preserving user content (SOUL.md, MEMORY, USER PROFILE, skills catalog)
5. **Unused Skills Removal** — Strips configurable list of unused skills from `<available_skills>` XML to reduce body size
6. **Property Renaming** — Renames framework-specific schema properties
7. **CC Tool Stubs** — Injects Hermes Agent tool stubs to match expected tool set
8. **Stainless SDK Headers** — Full Hermes Agent identity headers
9. **Thinking Block Protection** — Masks `thinking`/`redacted_thinking` blocks before transforms to preserve byte-equality

**Inbound (response to Hermes):**
10. **Full Reverse Mapping** — Restores all tool names, property names, and identifiers in both SSE streaming and JSON responses
11. **Event-Aware SSE** — Processes per SSE event with thinking block passthrough

## Requirements

- Node.js 18+
- Claude Max or Pro subscription
- Hermes Agent CLI installed and authenticated (`claude auth login`)

## Quick Start

```bash
git clone https://github.com/avaclaw1/hermes-billing-proxy.git
cd hermes-billing-proxy
node proxy.js
```

The proxy starts on port `18802` by default. Verify with:

```bash
curl http://127.0.0.1:18802/health
```

## Hermes Configuration

Point Hermes at the proxy by editing your `~/.hermes/config.yaml`:

### Required Settings

```yaml
# Route all API traffic through the proxy
model:
  default: claude-opus-4-6        # or claude-sonnet-4, etc.
  provider: anthropic
  base_url: http://127.0.0.1:18802
  api_key: dummy                   # proxy handles auth, this is ignored

providers:
  anthropic:
    base_url: http://127.0.0.1:18802
    api_key: dummy

# Disable fallback providers — they'd bypass the proxy
fallback_providers: []
```

### Recommended Settings

These aren't strictly required but improve reliability and reduce detection surface:

```yaml
# Enable prompt caching (reduces repeated tokens → fewer detection signals)
prompt_caching:
  enabled: true

# Streaming must stay enabled (proxy handles SSE)
streaming:
  enabled: true

# Extended thinking works normally through the proxy
agent:
  reasoning_effort: high
```

### Delegation / Subagents

If you use Hermes's delegation feature (subagents), they inherit the main provider config automatically. No extra config needed — subagent requests route through the same proxy.

If you've overridden delegation to use a different provider, make sure it also points to the proxy:

```yaml
delegation:
  provider: anthropic
  base_url: http://127.0.0.1:18802
  api_key: dummy
```

### Custom Providers (Not Proxied)

Any custom providers defined outside the `anthropic` block are **not** routed through the proxy. This is useful if you want some models (e.g., local vLLM, Gemini via OpenRouter) to bypass the proxy entirely:

```yaml
custom_providers:
  - name: vllm
    base_url: http://127.0.0.1:8080/v1
    api_key: ""
    api_mode: chat_completions
```

### Compression / Auxiliary Models

Hermes uses auxiliary models for compression, vision, session search, etc. These use their own provider config. If they're set to `provider: auto`, they'll pick up the main anthropic config and route through the proxy — which is fine. If you'd rather save proxy bandwidth, point them at a cheaper provider directly:

```yaml
# Example: use Gemini for compression instead of routing through proxy
compression:
  enabled: true
  summary_model: google/gemini-3-flash-preview
  summary_provider: auto     # uses OpenRouter, bypasses proxy

auxiliary:
  compression:
    provider: auto            # won't hit the proxy if summary_provider is set
```

## Proxy Configuration

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

The proxy removes skills listed in the `REMOVE_SKILLS` set to reduce request body size. Edit this set in `proxy.js` to customize. Smaller bodies are less likely to trigger composite scoring.

### Token Refresh

The proxy automatically refreshes the Hermes Agent OAuth token when it's within 2 minutes of expiry:

1. Triggers `claude -p "ping"` to refresh the credential store
2. Re-reads the refreshed token from `~/.claude/.credentials.json`
3. Retries with exponential backoff (15s → 30s → 60s... capped at 10m)
4. Gives up after 20 consecutive failures

No cron job needed.

## Systemd Service (Linux)

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/hermes-proxy.service << 'EOF'
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

Replace `/path/to/proxy.js` with your actual path (e.g., `/home/user/.claude-ws/billing-proxy/proxy.js`).

Check status:
```bash
systemctl --user status hermes-proxy.service
journalctl --user -u hermes-proxy.service -f
```

## Troubleshooting

### 400 "extra usage" errors
The API is rejecting requests as non-Claude-Code. Common causes:
- **Token expired** — Check proxy logs for refresh errors. Run `claude auth login` manually.
- **New detection layer** — Anthropic may have added new fingerprinting. Check if the upstream [openclaw proxy](https://github.com/zacdcook/openclaw-billing-proxy) has updates.
- **Body too small** — Very short conversations (< 20KB) have fewer signals to mask. Usually resolves after a few turns.

### Proxy not starting
- Ensure port 18802 isn't already in use: `lsof -i :18802`
- Check Node.js version: `node --version` (need 18+)
- Check Hermes Agent credentials exist: `ls ~/.claude/.credentials.json`

### Hermes can't connect
- Verify `base_url` in config.yaml matches the proxy port
- Ensure the proxy is running: `curl http://127.0.0.1:18802/health`
- If using WSL, `127.0.0.1` works — don't use `localhost` if DNS resolution is flaky

## How Detection Works

Anthropic uses multiple layers to detect non-Claude-Code clients:

1. **Billing header** — Checks for `x-anthropic-billing-header` with valid Hermes Agent fingerprint
2. **String triggers** — Scans for known framework-specific phrases
3. **Tool name fingerprint** — Identifies the framework by its combination of tool names
4. **System prompt template** — Matches known boilerplate text blocks (exact/fuzzy matching against published source code)
5. **Body scoring** — Composite score across all signal sources

The proxy addresses all five layers. The key insight: detection matches against **exact boilerplate text blocks** from source code, not individual keywords. Short paraphrases with the same semantic meaning pass because they don't match the known templates.

## License

MIT
