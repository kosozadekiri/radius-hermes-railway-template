# Hermes Agent Railway Template

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/hermes-railway-template?referralCode=uTN7AS&utm_medium=integration&utm_source=template&utm_campaign=generic)

Deploy [Hermes Agent](https://github.com/NousResearch/hermes-agent) to Railway as a worker service with persistent state.

This template is worker-only: setup and configuration are done through Railway Variables, then the container bootstraps Hermes automatically on first run.

## What you get

- Hermes gateway running as a Railway worker
- First-boot bootstrap from environment variables
- Persistent Hermes state on a Railway volume at `/data`
- Telegram, Discord, or Slack support (at least one required)
- Built-in Radius Testnet wallet (instantiated via radius-cli on first boot, auto-funded via faucet)
- Agent discovery layer served at `/.well-known/*` — ERC 8004 registration, Cloudflare agent skills discovery, and A2A agent card
- Agent-to-agent (A2A) communication with two execution modes: direct (inline `message/send` + `message/stream`) and delegated (webhook-backed async submission)
- Persistent cryptographic identity derived from the radius-cli wallet keystore; exported key material is used for JWT and on-chain signing paths
- Built-in discovery aggregation tool via `get_agent_info`
- Built-in deterministic ERC-8004 registry tools for reading and writing Radius agent registrations
- Built-in outbound A2A helper via `send_a2a_message` with sender-side correlation logging
- Railway-friendly observability: structured JSON logs from the agent server plus forwarded Hermes harness log files

## How it works

1. You configure required variables in Railway.
2. On first boot, entrypoint initializes Hermes under `/data/.hermes`.
3. On future boots, the same persisted state is reused.
4. Container starts the Python/FastAPI agent server and `hermes gateway` in parallel.

## Quick start (deploy from CLI)

If you're deploying manually with the Railway CLI:

```bash
# 1. Create a Railway project and add a volume mounted at /data
# 2. Link this repo to the project
railway link

# 3. Set required env vars (at minimum: a provider + a platform)
railway variables --set ANTHROPIC_API_KEY=sk-ant-...
railway variables --set TELEGRAM_BOT_TOKEN=123456:ABC...

# 4. Run the pre-deploy check, then deploy
./deploy.sh
```

`deploy.sh` validates that the required env vars are set in your linked Railway project before uploading anything, so you get a clear error locally instead of a crash loop in production.

If you want a full clean slate, run:

```bash
./deploy.sh --reset-state
```

That clears the persisted Railway volume paths used by Hermes before deploying:

- `/data/.hermes`
- `/data/workspace`
- `/data/.claude`

This resets agent memory, sessions, pairing state, ByteRover state, workspace files, and the persisted Radius wallet.

## Example prompts

As soon as the agent is live, these are good first prompts to try in chat.

The bundled public Radius-facing skills include the template-owned skills plus any vendored upstream Radius marketplace skills that are present in the deployed image and marked `published: true`:

- `radius-wallet`
- `a2a-comms`
- `registering-agent`

`radius-wallet`, `a2a-comms`, and `registering-agent` are template-owned. Additional Radius marketplace skills are sourced from the vendored upstream Radius skills repo at deploy time and retain the upstream authorship metadata.

### Radius wallet and funding

- *"What is my wallet address?"*
- *"Check my Radius wallet balance."*
- *"How much SBC and RUSD do I have right now?"*
- *"Show me my wallet address and give me the testnet explorer link."*
- *"Do I already have testnet funds, or do I need to use the faucet?"*
- *"How do I get more Radius testnet funds?"*

### Radius transactions

- *"Send 0.001 SBC to 0x1234... and show me the transaction hash."*
- *"Before sending, tell me if I have enough balance to send 5 SBC."*
- *"What would happen if I tried to send more SBC than I have?"*
- *"Check the status of this Radius transaction: 0xabc..."*

### Radius developer questions

- *"What is Radius, and what can this agent do with it?"*
- *"Give me the Radius Testnet chain ID, RPC URL, and explorer."*
- *"How is Radius different from Ethereum for app developers?"*
- *"What fee assumptions should I avoid when building on Radius?"*
- *"Show me the correct network settings for Radius Testnet and mainnet."*

### Agent-to-agent workflows

- *"What is this agent's DID?"*
- *"Show me this agent's public discovery information."*
- *"What can another A2A agent learn from this agent card?"*
- *"Send a task to https://<other-agent>/a2a asking it to introduce itself."*
- *"Use the outbound A2A tool to ask the peer agent what skills it has."*
- *"Continue the existing A2A conversation with the peer agent and ask for a status update."*

### ERC-8004 registration workflows

- *"Show me the current ERC-8004 registry stats on Radius testnet."*
- *"Read the registration for agent 0 on Radius testnet."*
- *"List all registered agents on Radius testnet."*
- *"Register this agent on ERC-8004 using the current wallet and DID."*
- *"Update agent 2's ERC-8004 registration with a new DID and services map."*
- *"Patch an existing registration while preserving current metadata."*
- *"Add canonical web/A2A/DID aliases plus a GoDaddy ANS pointer."*

### Payments between agents

- *"Ask the peer agent for its wallet address."*
- *"Send Agent 2 a small amount of SBC on testnet and tell me the tx hash."*
- *"Delegate a task to the peer agent, then summarize the A2A correlation ids you used."*
- *"Explain how an A2A task id, message id, and context id relate to each other here."*

### Memory and operator context

- *"What durable things can you remember between sessions?"*
- *"Remember that this wallet belongs to the demo operator."*
- *"Record this transaction and describe why it happened."*

### Optional Linear prompts

If `LINEAR_API_KEY` is set, these are useful immediately:

- *"List my Linear teams."*
- *"Show my current Linear projects."*
- *"Create a Linear issue for improving Railway observability."*
- *"Summarize open issues related to A2A or logging."*

## Railway deploy instructions

In Railway Template Composer:

1. Add a volume mounted at `/data`.
2. Deploy as a worker service.
3. Set only the variables you actually need (see below).

Template defaults (already included in `railway.toml`):

- `HERMES_HOME=/data/.hermes`
- `HOME=/data`
- `MESSAGING_CWD=/data/workspace`
- `LLM_MODEL=openai/gpt-5.4-nano`

## Important: how to set variables in Railway

**Only add variables you intend to use. Do not add optional variables with empty values.**

Railway injects every variable you define into the container environment, even if the value is empty. Hermes parses several variables as integers (e.g. `HERMES_MAX_ITERATIONS`, `TERMINAL_TIMEOUT`)

**Right way:** add only the variables you need, with real values.

**Wrong way:** copy the full `.env.example` into Railway with all optional fields left blank.

If you want to use `.env.example` as a reference, only add the variables you plan to fill in. Leave everything else out of Railway entirely.

## Secure Railway: pin & fresh-install checklist

Follow these steps to deploy safely and ensure a fresh/clean initialization on a new `/data` volume:

1. Add a persistent volume in Railway and mount it at `/data` (this repository's `railway.toml` requires it). The entrypoint initializes Hermes under `/data/.hermes` on first boot.

2. BEFORE you deploy, set the build-time pin to the exact Hermes commit or tag we recommend:

   - HERMES_GIT_REF=3c27eb6

   Important: Railway makes build-time variables available at image build. Set this variable in Railway Console *before* running `./deploy.sh` or `railway up` so the image is built from the pinned commit.

3. Set at least one inference provider and one messaging platform (only include variables that have real values):

   - OPENROUTER_API_KEY=sk-or-...
   - TELEGRAM_BOT_TOKEN=123456:ABC...  (or DISCORD_BOT_TOKEN / SLACK_BOT_TOKEN + SLACK_APP_TOKEN)
   - PUBLIC_URL=https://<your-railway-domain>  (recommended for DID & discovery)
   - TELEGRAM_ALLOWED_USERS=123456789  (strongly recommended to restrict access)

4. (Optional) If you want to force a full fresh initialization (delete any persisted state on the volume) use the pre-deploy helper locally before running deploy:

   ```bash
   ./deploy.sh --reset-state
   ```

   This clears `/data/.hermes`, `/data/workspace`, and `/data/.claude` via Railway SSH before building/deploying. Use with caution — it irreversibly removes persisted state.

5. Run the repository's pre-deploy sanity check and deploy:

   ```bash
   ./deploy.sh
   ```

   The script will abort with clear errors if a required provider or messaging token is missing, or if the Railway project has not mounted a required `/data` volume.

6. After deploy, check logs and the homepage/discovery endpoints to confirm success. Example checks:

   - Railway logs for `[bootstrap]` messages
   - Visit `https://<your-railway-domain>/` and `https://<your-railway-domain>/.well-known/agent-card.json`
   - Use Railway SSH to run `hermes status` inside the container if needed

Notes:
- Do not run `hermes update` inside Railway — to update Hermes, change `HERMES_GIT_REF` to a new tag/SHA and redeploy (run `hermes config migrate` over Railway SSH if upstream adds new config options).
- The `deploy.sh` helper already fails fast for missing providers, missing messaging tokens, and missing `/data` mount — use it to avoid crash loops.

## Required runtime variables

Set at least one inference provider:

| Variable | Description |
|---|---|
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `OPENAI_API_KEY` + `OPENAI_BASE_URL` | OpenAI-compatible provider |
| `ANTHROPIC_API_KEY` | Anthropic direct API |

Set at least one messaging platform:

| Variable | Description |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token from @BotFather |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` | Slack (both required) |

## Discord bot setup

If you're using Discord, you need to create a bot application and enable the correct intents before the token will work.

### 1. Create the bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications) and click **New Application**.
2. Give it a name, then open the **Bot** tab on the left sidebar.
3. Click **Reset Token** to generate your bot token — copy it now (you won't see it again without resetting). This goes in `DISCORD_BOT_TOKEN`.

### 2. Enable privileged intents

Still on the **Bot** tab, scroll down to **Privileged Gateway Intents** and enable:

- **Server Members Intent** — required for the bot to see guild members
- **Message Content Intent** — required for the bot to read message content (without this, Hermes receives empty messages)

Click **Save Changes**. Skipping this step is the most common reason Discord bots connect but never respond.

### 3. Invite the bot to your server

1. Go to the **OAuth2 → URL Generator** tab.
2. Under **Scopes**, check `bot`.
3. Under **Bot Permissions**, check at minimum: `Send Messages`, `Read Message History`, `View Channels`.
4. Copy the generated URL at the bottom and open it in your browser.
5. Select your server and click **Authorize**.

### 4. Get your Discord user ID

To populate `DISCORD_ALLOWED_USERS`, you need your Discord user ID (a large integer, not your username):

1. In Discord, go to **Settings → Advanced** and enable **Developer Mode**.
2. Right-click your name anywhere in Discord and select **Copy User ID**.

Use that value in Railway:
```
DISCORD_ALLOWED_USERS=123456789012345678
```

---

## Recommended variables

### Allowlists (strongly recommended)

Restrict access to specific user IDs. Format: plain comma-separated integers, no quotes, no brackets.

```
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=123456789012345678,234567890123456789
SLACK_ALLOWED_USERS=U01234ABCDE,U09876WXYZ
```

To find your Telegram user ID, message [@userinfobot](https://t.me/userinfobot).

### Provider selection

If you set multiple provider keys, pin which one Hermes uses:

```
HERMES_INFERENCE_PROVIDER=openrouter
```

Without this, Hermes auto-selects and may not pick the one you expect.

### Model override

```
LLM_MODEL=openai/gpt-5.4-nano
```

Use any model ID supported by your provider. OpenRouter model IDs look like `openai/gpt-5.4-nano` or `openai/gpt-4o`.

## Radius wallet

This template includes a built-in Radius Testnet wallet. On first boot, the entrypoint:

1. Instantiates a local radius-cli keystore (or reuses an existing one) under `${RADIUS_HOME:-/data/.hermes/.radius-cli}`.
2. Caches the wallet address for runtime metadata and homepage display.
3. Exports key material for JWT + ERC-8004 signing compatibility.
4. Requests SBC testnet tokens from the Radius faucet (unless `RADIUS_AUTO_FUND=false`).

The agent can then check balances, send SBC tokens, and show explorer links — all via natural language in chat.

The bundled wallet tools are backed by `radius-cli` and use a local persistent keystore under `${RADIUS_HOME:-/data/.hermes/.radius-cli}`.

- Wallet creation happens automatically on first boot via `radius-cli wallet address`.
- The same local wallet is used for wallet actions, DID/JWT auth, homepage wallet summary, and ERC-8004 identity.

## ERC-8004 registry tools

This template now includes a bundled `erc8004-registry` plugin plus a lightweight `registering-agent` skill.

Use this interface for ERC-8004 work instead of temporary scripts. The plugin exposes deterministic tools for:

- reading one registration
- listing live registrations from the registry contract
- inspecting registry stats
- registering the current agent from defaults
- registering a new agent
- updating an existing agent URI with a complete replacement registration
- patching an existing registration while preserving current metadata
- adding canonical web/A2A/DID aliases plus a GoDaddy ANS pointer

The plugin ships with checked-in Radius network constants for `testnet` and `mainnet`. `testnet` is enabled now and uses the deployed registry at `0x5cd923Ce1244d5498Bf3f9E0F3a374C2567F1A31` on c... (truncated for brevity in commit)
