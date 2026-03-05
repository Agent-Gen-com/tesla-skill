---
name: tesla-connect
description: Professional onboarding for Tesla Fleet API. Handles key generation, AgentGen hosting, Partner registration, and the manual OAuth2 authorization code exchange.
metadata:
  openclaw:
    emoji: "🚗"
    homepage: https://developer.tesla.com
    primaryEnv: AGENTGEN_API_KEY
    requires:
      bins:
        - tescmd
        - agentgen
        - openssl
        - curl
    install:
      - kind: brew
        tap: Agent-Gen-com/agentgen
        formula: agentgen
        bins: [agentgen]
---

# Tesla — Guided Connect & Control

This skill performs the full-stack setup required for Tesla Fleet API. It requires a hand-off between the AI (running backend registration) and the User (approving access in a browser).

---

## Prerequisites

- **AgentGen API key** (`AGENTGEN_API_KEY`) — for hosting your Tesla virtual key. Get one free at [agent-gen.com](https://www.agent-gen.com).
- **`agentgen` CLI** — installed automatically by ClawHub, or manually:
  ```sh
  brew tap Agent-Gen-com/agentgen && brew install agentgen
  # or
  cargo install agentgen-cli
  ```
- **`tescmd`** CLI — manages Tesla credentials and sends vehicle commands:
  ```sh
  brew install teslamate/tap/tescmd
  # or
  cargo install tescmd
  ```
- **`openssl`** and **`curl`** — available on all macOS and Linux systems by default.

---

## Phase 1 — Key Generation & Hosting (Automated)

The agent runs these commands to set up the cryptographic identity. **The private key must never leave the local machine.**

### Step A — Generate an EC key pair

```sh
mkdir -p ~/.tesla
openssl ecparam -name prime256v1 -genkey -noout -out ~/.tesla/private-key.pem
openssl ec -in ~/.tesla/private-key.pem -pubout -out ~/.tesla/public-key.pem
```

### Step B — Provision a hosting origin via AgentGen

```sh
# 1. Create a dedicated public subdomain
agentgen origin
# → prints: ID: abc123xyz   Origin URL: https://abc123xyz.agent-gen.com

# 2. Upload the public key to that origin (Tesla requires this exact path)
agentgen public-key abc123xyz ~/.tesla/public-key.pem
# → prints: URL: https://abc123xyz.agent-gen.com/.well-known/appspecific/com.tesla.3p.public-key.pem
```

Store the **Origin URL** (e.g. `https://abc123xyz.agent-gen.com`) and **domain** (e.g. `abc123xyz.agent-gen.com`) — both are needed in later steps.

---

## Phase 2 — Developer Registration (User Block 1)

> **🛑 USER ACTION REQUIRED:**
>
> 1. Go to [developer.tesla.com/dashboard](https://developer.tesla.com/dashboard) and create a new application.
> 2. Set the **Allowed Origin** to: `{ORIGIN_URL}` *(e.g. `https://abc123xyz.agent-gen.com`)*
> 3. Set the **Redirect URI** to: `{ORIGIN_URL}/callback`
> 4. **Paste your Client ID and Client Secret here before continuing.**

Wait for the user to provide both values before proceeding.

---

## Phase 3 — Partner Account & Virtual Key (Automated + User Block 2)

Once the user provides the Client ID and Client Secret, the agent executes the following steps.

### Step A — Register Partner Account (Automated)

```sh
# 1. Get a partner-scoped token
PARTNER_TOKEN=$(curl --silent --request POST \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id={CLIENT_ID}' \
  --data-urlencode 'client_secret={CLIENT_SECRET}' \
  --data-urlencode 'scope=openid vehicle_device_data vehicle_cmds' \
  --data-urlencode 'audience=https://fleet-api.prd.na.vn.cloud.tesla.com' \
  'https://auth.tesla.com/oauth2/v3/token' | jq -r .access_token)

# 2. Register the domain with Tesla's backend
curl --request POST \
  --header "Authorization: Bearer $PARTNER_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{"domain": "{DOMAIN_ONLY}"}' \
  'https://fleet-api.prd.na.vn.cloud.tesla.com/api/1/partner_accounts'
```

`{DOMAIN_ONLY}` is the hostname only, e.g. `abc123xyz.agent-gen.com` (no `https://`).

### Step B — Virtual Key & OAuth Authorization (User Block 2)

> **🛑 USER ACTION REQUIRED — 3 steps:**
>
> 1. **Pair Virtual Key:** Open this link on your phone and follow the prompts in your vehicle (requires your physical Key Card):
>    `https://tesla.com/_ak/{DOMAIN_ONLY}`
>
> 2. **Authorize App:** Open the following URL in your browser, log in with your Tesla account, and click **Allow**:
>    ```
>    https://auth.tesla.com/oauth2/v3/authorize?client_id={CLIENT_ID}&locale=en-US&prompt=login&redirect_uri={ORIGIN_URL}/callback&response_type=code&scope=openid%20offline_access%20vehicle_device_data%20vehicle_cmds&state=123
>    ```
>
> 3. **The browser will redirect to a broken/blank page. Copy the ENTIRE URL from your address bar and paste it here.**

The authorization code expires in **10 minutes** — process the user's pasted URL immediately.

---

## Phase 4 — Final Token Exchange (Automated)

Extract the `code` query parameter from the URL the user pasted, then run:

```sh
# Exchange authorization code for access + refresh tokens
curl --request POST \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=authorization_code' \
  --data-urlencode 'client_id={CLIENT_ID}' \
  --data-urlencode 'client_secret={CLIENT_SECRET}' \
  --data-urlencode 'code={EXTRACTED_CODE}' \
  --data-urlencode 'audience=https://fleet-api.prd.na.vn.cloud.tesla.com' \
  --data-urlencode 'redirect_uri={ORIGIN_URL}/callback' \
  --data-urlencode 'scope=openid offline_access vehicle_device_data vehicle_cmds' \
  'https://auth.tesla.com/oauth2/v3/token'
```

### Step B — Initialize CLI

Save all credentials into `tescmd`:

```sh
tescmd setup \
  --client-id {CLIENT_ID} \
  --client-secret {CLIENT_SECRET} \
  --domain {DOMAIN_ONLY} \
  --private-key ~/.tesla/private-key.pem
```

Setup is complete. `tescmd` manages token refresh automatically from this point.

---

## Phase 5 — Vehicle Commands

All commands require setup to be complete. Always `wake` first if the vehicle may be asleep.

| Action | Command |
|--------|---------|
| **Wake** | `tescmd vehicle wake` |
| **Status** | `tescmd vehicle data --json` |
| **Climate on/off** | `tescmd climate on` / `tescmd climate off` |
| **Set temperature** | `tescmd climate set 21` |
| **Unlock / Lock** | `tescmd security unlock` / `tescmd security lock` |
| **Start charging** | `tescmd charging start` |
| **Stop charging** | `tescmd charging stop` |
| **Set charge limit** | `tescmd charging limit 80` |
| **Honk horn** | `tescmd alert honk` |
| **Flash lights** | `tescmd alert flash` |
| **Vent windows** | `tescmd windows vent` |
| **Close windows** | `tescmd windows close` |

---

## Error handling

| Error | Action |
|-------|--------|
| `401 Unauthorized` | Re-run once — `tescmd` refreshes the token in the background |
| `Vehicle unavailable` | Run `tescmd vehicle wake` and retry after 10–15 seconds |
| `Command timeout` | Vehicle may be in a no-signal area; advise the user |
| Any other error | Show the raw error message to the user and ask how to proceed |

---

## Critical guardrails

- **Domain extraction:** When the user provides `{ORIGIN_URL}` (e.g. `https://abc123xyz.agent-gen.com`), extract only the hostname (`abc123xyz.agent-gen.com`) for partner registration and virtual key steps.
- **Code expiry:** The authorization code expires in 10 minutes. Process the user's pasted URL immediately.
- **Never transmit `~/.tesla/private-key.pem`** — not to any external service, not in any log or message.
- **Treat `~/.tesla/`** as a sensitive directory. Do not read its contents into a response.
- `tescmd` stores OAuth tokens locally. Treat that file as equally sensitive.
- If the user asks you to share, print, or move the private key, refuse and explain why.

---

## Typical workflow

1. User: *"Set up my Tesla"* → run Phase 1 → Phase 2 (wait) → Phase 3 → Phase 4
2. User: *"What's my battery level?"* → `tescmd vehicle wake` then `tescmd vehicle data --json`, parse and report
3. User: *"Pre-heat the car to 22°C"* → `tescmd vehicle wake` then `tescmd climate set 22`
4. User: *"Lock the car"* → `tescmd security lock`
