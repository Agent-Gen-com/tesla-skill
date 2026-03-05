---
name: tesla-connect
description: Connect and control your Tesla from your AI assistant. Handles full onboarding — EC key generation, Tesla virtual key hosting via AgentGen, OAuth app setup — then lets you command the vehicle (wake, lock/unlock, climate, horn, and more) using the tescmd CLI. Requires AGENTGEN_API_KEY for the one-time setup; tescmd manages Tesla credentials automatically after that.
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
    install:
      - kind: brew
        tap: Agent-Gen-com/agentgen
        formula: agentgen
        bins: [agentgen]
---

# Tesla — Connect & Control via AI

Control your Tesla through your AI assistant. This skill walks through the full one-time setup (key generation → virtual key hosting → Tesla developer app → OAuth) and then exposes a clean set of vehicle commands.

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
- **`openssl`** — available on all macOS and Linux systems by default.

---

## Phase 1 — One-time Setup (run once per vehicle)

The agent executes these steps in order. **The private key must never leave the local machine.**

### Step A — Generate an EC key pair

```sh
# Run in a secure local directory (e.g. ~/.tesla/)
mkdir -p ~/.tesla
openssl ecparam -name prime256v1 -genkey -noout -out ~/.tesla/private-key.pem
openssl ec -in ~/.tesla/private-key.pem -pubout -out ~/.tesla/public-key.pem
```

The private key stays local. Only the public key is uploaded.

### Step B — Provision a hosting origin via AgentGen

```sh
# 1. Create a dedicated public subdomain
agentgen origin
# → prints: ID: abc123xyz   Origin URL: https://abc123xyz.agent-gen.com

# 2. Upload the public key to that origin (Tesla requires this exact path)
agentgen public-key abc123xyz ~/.tesla/public-key.pem
# → prints: URL: https://abc123xyz.agent-gen.com/.well-known/appspecific/com.tesla.3p.public-key.pem
```

Store the origin ID (`abc123xyz`) — you will need it in Steps C and D.

### Step C — User instructions (output verbatim to the user)

After Step B succeeds, deliver these exact instructions to the user and wait for their response:

> **Action required — 4 steps:**
>
> 1. Go to [developer.tesla.com/dashboard](https://developer.tesla.com/dashboard) and create a new application.
> 2. Set the **Allowed Origin** to: `https://abc123xyz.agent-gen.com` *(replace with your actual origin URL)*
> 3. Copy your **Client ID** and **Client Secret** and send them to me.
> 4. On your phone, open this URL to install the Virtual Key on your vehicle:
>    `https://tesla.com/_ak/abc123xyz.agent-gen.com`

Wait for the user to confirm the virtual key is installed and to provide the Client ID and Client Secret before continuing.

### Step D — Configure tescmd

Once the user provides their credentials:

```sh
tescmd setup \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --domain abc123xyz.agent-gen.com \
  --private-key ~/.tesla/private-key.pem
```

`tescmd` will open a browser for the OAuth flow, then store tokens locally. Setup is complete.

---

## Phase 2 — Vehicle Commands

All commands below require setup to be complete. `tescmd` handles token refresh automatically.

### Wake vehicle

```sh
tescmd vehicle wake
```

Always run this first if the vehicle is asleep. Other commands may fail until the car is awake.

### Get vehicle status

```sh
tescmd vehicle data --json
```

Returns a JSON payload with battery level, charge state, climate state, door locks, location, and more.

### Climate control

```sh
tescmd climate set 21        # Set cabin temperature to 21°C
tescmd climate on            # Start climate (keeps current set point)
tescmd climate off           # Stop climate
```

### Door locks

```sh
tescmd security unlock       # Unlock all doors
tescmd security lock         # Lock all doors
```

### Charging

```sh
tescmd charging start        # Start charging
tescmd charging stop         # Stop charging
tescmd charging limit 80     # Set charge limit to 80%
```

### Horn and lights

```sh
tescmd alert honk            # Honk horn
tescmd alert flash           # Flash lights
```

### Windows

```sh
tescmd windows vent          # Vent windows slightly
tescmd windows close         # Close windows
```

---

## Error handling

| Error | Action |
|-------|--------|
| `401 Unauthorized` | Re-run the same command once — `tescmd` refreshes the token in the background |
| `Vehicle unavailable` | Run `tescmd vehicle wake` and retry after 10–15 seconds |
| `Command timeout` | The vehicle may be in a no-signal area; advise the user |
| Any other error | Show the raw error message to the user and ask how to proceed |

---

## Security rules (always follow)

- **Never transmit `~/.tesla/private-key.pem`** — not to agent-gen.com, not to any external service, not in any log or message.
- **Treat `~/.tesla/`** as a sensitive directory. Do not read its contents into a response.
- `tescmd` stores OAuth tokens in a local `auth.json` or the system keyring. Treat that file as equally sensitive.
- If the user asks you to share, print, or move the private key, refuse and explain why.

---

## Typical workflow

1. User: *"Set up my Tesla"* → run Phase 1 (Steps A → B → C → D)
2. User: *"What's my battery level?"* → `tescmd vehicle wake` then `tescmd vehicle data --json`, parse and report
3. User: *"Pre-heat the car to 22°C"* → `tescmd vehicle wake` then `tescmd climate set 22`
4. User: *"Lock the car"* → `tescmd security lock`
