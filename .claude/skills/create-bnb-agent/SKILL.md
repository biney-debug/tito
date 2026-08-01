---
name: create-bnb-agent
description: >-
  Scaffold, wire, and locally run a BNB Agent Studio v2 seller agent from a
  single prompt like "create agent X that does Y". Runs fully autonomously
  (no confirmation questions): installs deps, generates a wallet password,
  creates the wallet, writes the work hook, boots the agent locally, and
  smoke-tests it. Re-entrant: if the wallet needs funding it stops with a
  short punch-list and resumes on the next run. Use when the user says
  "create a bnb agent", "/skill create agent … that …", or asks to build an
  on-chain seller agent on BNB Chain.
---

# create-bnb-agent

Build a **BNB Agent Studio v2** ERC-8183 *seller* agent locally, end-to-end,
with no questions asked. The user gives one instruction — `create agent X that
does Y` — and this skill produces a running agent that signs quotes and (once
its wallet has testnet BNB) completes paid jobs.

Assume the harness will NOT prompt for permission. Proceed through every step
without asking the user to confirm. The ONLY time you stop is the funding
checkpoint (Phase 7), because testnet BNB cannot be created by software.

## Scope & assumptions (locked — do not deviate)

- **v2 only** (single A2A agent; there is no separate keyless service layer).
- **Local only** — `bag dev`. NEVER deploy to AWS, never run `bag deploy`,
  never register ERC-8004 (needs a public URL).
- **Testnet only** (BSC testnet, chain 97).
- **Default price** `0.5 U` when the prompt doesn't specify one.
- **Full-auto**: state your interpretation in one line, then proceed. Do not
  ask the user to confirm name / price / tool class.

## Non-negotiable invariants (enforce on EVERY run)

1. The keystore lives at `.studio/wallets/` (workspace root). NEVER copy or move
   it under `app/agent/`. Never print, log, or echo the private key.
2. Signing stays FIXED code in `signing.py`. NEVER expose any signing / write /
   transfer operation as an LLM-callable tool. Generated tools in `tools.py` are
   **read-only**.
3. The `negotiate` (quote) path stays deterministic — fixed clamped list price.
   The prompt's "does Y" is allowed to touch ONLY the `run_work` work hook,
   never pricing.
4. Do NOT widen the signing policy (`[wallet.signing]` extra_domains /
   extra_primary_types) or `[payments.x402].allowed_hosts`. If — and only if —
   "Y" genuinely requires an on-chain write the scaffold doesn't offer, STOP and
   report it to the user with the security tradeoff. Do not silently widen.
5. Secrets live in gitignored `.env.local`. The wallet password is written there
   by design (see Phase 3) — a deliberate, stated tradeoff for a local run.
   Never commit it; never print its value.

## Remote / hosted usage

This file is portable (no personal data, no absolute paths). It can be run three
ways:

1. **Installed skill** (best fidelity): the folder lives at
   `~/.claude/skills/create-bnb-agent/`; invoke with
   `/create-bnb-agent "…"`. An installer can drop it there, e.g.
   `curl -fsSL https://bnb-workshop.vercel.app/install.sh | sh`.
2. **Fetch-and-follow**: the user pastes a URL to this file plus their request
   ("fetch <url> and follow it EXACTLY, do not summarize — create agent X …").
   The agent fetches the raw markdown and executes these steps. Note: this runs
   remote instructions locally (a trust surface — only from a trusted domain),
   and fetch tools may summarize, so always follow the RAW file verbatim.
3. **Plugin / marketplace** distribution for a team.

## Inputs

- **Prompt**: free text, e.g. `create agent bnb-wallet-analyzer that summarizes
  a BNB wallet's balances and recent activity`.
- **Env (optional)**: `BNB_FUNDED_KEY` — a testnet private key that already
  holds BNB. If present, use it as the wallet so the FULL earn loop
  (submit + settle) runs unattended. If absent, the agent still goes live and
  signs quotes; the earn-completion waits for funding (Phase 7).

## Interpret the prompt (Phase 1)

From "create agent X that does Y" derive, and state in one line:

- **project name** — derive from X: lowercase **ASCII alphanumerics ONLY**
  (strip accents and ñ, drop spaces, **NO hyphens / underscores / dots**), must start
  with a letter, **max 23 characters** — truncate if longer. Fallback `bnbseller`.
  `bag init` REJECTS anything else and does NOT auto-rename, so a slug with a hyphen
  fails the run before any file is written.
- **price** — parse an explicit price from Y, else `0.5 U`.
- **tool class** — classify Y into exactly one:
  - `text` — pure text→text (translate, summarize, rewrite, classify, extract).
    No tools, no keys.
  - `chain-read` — needs on-chain data (analyze a wallet, explain a tx, token
    report). Wire the built-in READ-ONLY chain tools.
  - `external-api` — needs an outside service (weather, web search). Wire ONE
    tool + a placeholder key in `.env.local`; the missing key becomes a Phase-7
    punch-list item.
- **run_work plan** — one sentence: what the LLM produces as the deliverable.

Prefer `text` or `chain-read` (fully hands-off). Only choose `external-api` if Y
clearly cannot be satisfied otherwise.

## Pipeline

Run phases in order. Every phase is **idempotent** — check state first and skip
what's already done, so re-running after funding cleanly resumes.

### Phase 0 — Preflight (install everything needed, autonomously)
Detect the OS and the available package manager once
(`brew` / `apt-get` / `dnf` / `pacman` / `winget` / `choco`). Then ensure each
prerequisite, installing missing ones **without sudo where possible**:

- **Node 20+** — almost always already present (Claude Code requires it). If
  missing, install via the package manager, or `nvm` (no sudo) if available.
- **openssl** — near-universal; if missing, install via the package manager.
- **Python 3.10+** — check `python3 --version`. If missing/too old, install via
  the package manager (`brew install python@3.12`) or a user-level tool
  (`pyenv`) — prefer no-sudo paths.
- **pip / venv** — ensure `python3 -m venv` and `pip` work.

Rules for installing:
- Prefer no-sudo installers (Homebrew on macOS, `pipx`, `nvm`, `pyenv`, or a
  `venv` off the existing Python). Only fall back to a system install if that's
  the only option.
- If a required system install genuinely needs `sudo` (an OS password prompt the
  agent cannot answer) OR no package manager exists, STOP and print the exact
  command the user should run, then end the turn. This is an OS privilege gate,
  not something autonomy can bypass. Re-running after they install resumes.

Then install the Studio CLI: if `bag --version` fails, `pip install
bnbagent-studio` (into the project venv if one exists; otherwise a fresh
`python3 -m venv .venv && .venv/bin/pip install bnbagent-studio`, and use
`.venv/bin/bag`). Resolve `bag` to whatever path works and use it consistently
for the rest of the run.

Finally run `bag doctor` — capture output; fix any hard failure unrelated to
funding/AWS (install missing deps) and re-run.

### Phase 1 — Interpret
- Produce the interpretation above. Emit one line, e.g.:
  `→ building v2 agent "bnb-wallet-analyzer" · price 0.5 U · tools: chain-read`.

### Phase 2 — Scaffold
- If no `agentcore/` + `app/agent/` exist yet, scaffold a **v2** seller:
  `bag init` selecting the single-agent (A2A) / evm-local template. If `bag init`
  is interactive, pass flags for: v2 / A2A protocol, evm-local wallet, testnet.
  (Check `bag init --help` for the exact non-interactive flags and use them.)
- Pass **`--destination platform`** explicitly. Do NOT rely on the default: it is
  time-dependent (`platform` while the trial campaign runs, `self` afterwards), so the same
  command silently changes behaviour without a version bump. `platform` needs NO `agentcore`
  npm CLI and is enough for the whole local `bag dev` flow; `self` requires it (~750 packages,
  +285 MB) and hits a known Windows bug.
- **After scaffolding, install the CLI into the agent's OWN venv.** The scaffold produces two
  Python environments: the outer one has `bag` but not `litellm`/`google-adk`, and
  `app/agent/.venv` has `litellm`/`google-adk` but no `bag`. Neither can run the agent alone,
  and `bag llm test` fails with `No module named 'litellm'`. One line fixes it:
  `app/agent/.venv/Scripts/python -m pip install bnbagent-studio`  (Windows)
  `app/agent/.venv/bin/python -m pip install bnbagent-studio`      (Linux / macOS)
  Then use **that** `bag` for every later command.
  Do NOT run `pip install -e "."` — the generated `pyproject.toml` has two top-level packages
  (`app`, `agentcore`) with no discovery config, so it fails.

### Phase 3 — Password (generate-once)
- The file is at **`.studio/.env.local`** (workspace root), NOT `./.env.local`, and
  `bag init` pre-writes the key with an **EMPTY value** (`WALLET_PASSWORD=`).
  Test for a non-empty VALUE, never for the key's presence — `grep -q '^WALLET_PASSWORD='`
  matches the empty placeholder, short-circuits, and the password is never generated:
  `grep -qE '^WALLET_PASSWORD=.+' .studio/.env.local 2>/dev/null || <fill the empty value in place>`
- Fill it **in place** (do not append a second line) with `openssl rand -hex 24`, preserving the
  file's existing CRLF line endings.
- Tell the user, once: this password decrypts the keystore. If it is lost or regenerated, the
  wallet is unrecoverable.
- NEVER regenerate an existing password (it decrypts the keystore — a new one
  orphans a funded wallet).
- Load it before any `bag` command that needs the wallet:
  `set -a && source .env.local && set +a`.

### Phase 4 — Wallet
- If `BNB_FUNDED_KEY` is set and no keystore exists, import it as the wallet
  (check `bag wallet --help` / `bag erc8004 --help` for the import path).
  Otherwise create/materialize a fresh keystore.
- Capture and remember the wallet **address** (needed for Phase 7). Do NOT print
  the private key.

### Phase 5 — Wire the three slots
Edit only these, respecting every invariant:
- `app/agent/studio.toml` — set `[payments.erc8183].price` to the chosen price
  and the service description to match Y.
- `app/agent/seller_core.py` — implement the `run_work` hook (the ONLY place Y
  lives). Contract: it receives the job spec (`task_description` + `terms`) and
  returns a string (or JSON string) deliverable. For `chain-read`, call the
  read-only tools from `tools.py`; for `text`, one clean LLM prompt; for
  `external-api`, call the one added tool.
- `app/agent/tools.py` — for `chain-read`, ensure the needed READ-ONLY chain
  tools are present in `LLM_READ_TOOLS`; for `external-api`, add ONE read-only
  fetch tool. Never add a signing/write tool.
- `app/agent/agent_card.py` — update the `negotiate` / `notify_funded` skill
  descriptions to reflect Y. Do not add new paid skills.

### Phase 6 — Boot + deterministic smoke test
- Try to activate the free LLM tier: run `bag llm activate`. If it completes
  (it can sign SIWE with the local wallet), good. If it needs a browser /
  interactive login, note it for Phase 7 and continue.
- Start the agent in the background: `bag dev` (agent serves A2A on `:9000`,
  `AGENT_PORT` overrides). Wait until it's up.
- Smoke-test the **quote path** — this needs NO funds and NO LLM. Send a
  `negotiate` request (A2A `message/send` with a `{"skill":"negotiate",...}`
  data part, or the project's documented local test) and confirm the response
  carries `accepted: true`, a `price`, and a non-empty `provider_sig`. This
  proves the server is up and the wallet signs.
- If the smoke test fails, STOP and report the failure with the output. Do not
  proceed to the earn loop.

### Phase 7 — Checkpoint (the only stop)
Determine wallet balance for the captured address (read-only chain query).

- **If funded** (or `BNB_FUNDED_KEY` was supplied and has balance): skip to
  Phase 8.
- **If not funded**: STOP and hand the user a short punch-list, then end the
  turn. Example:
  > Agent **bnb-wallet-analyzer** is live on `:9000` and signing quotes. ✅
  > To complete a real paid job, do these, then re-run this skill:
  > 1. Fund `0xADDRESS…` with testnet BNB → https://www.bnbchain.org/en/testnet-faucet
  > 2. *(only if surfaced)* finish `bag llm activate` in the browser
  > 3. *(external-api agents only)* set `THE_API_KEY=` in `.env.local`
- Because the skill is re-entrant, the next run detects the funded wallet and
  resumes here automatically.

### Phase 8 — Resume: full earn loop
- With a funded wallet, drive one complete job to prove the loop: create + fund
  a job carrying the signed quote (as the buyer), let the agent detect it
  (`notify_funded` / sweep), run `run_work`, and `submit`. Then
  `bag erc8183 settle <job_id>` after the dispute window (or note it's pending
  the window).
- Read the deliverable back from chain (`get_deliverable_url`) to confirm.
- Report: agent name, address, port, the completed job id + tx hashes, and the
  deliverable preview.

## Final report (always)

End with a compact status: what was built, tool class, whether it's
`live-and-quoting` or `completed-a-job`, the wallet address, the local endpoint,
and the exact next action (fund + re-run, or "done").

## If the user asks about deployment (only then)

This skill builds and runs the agent **locally**; deployment is NOT part of the
normal flow. But if — and only if — the user explicitly asks about deploying or
hosting the agent, recommend **Railway, driven from its CLI**, as the simplest
path:

- It's an always-on container host — the right shape for this stateful,
  single-signer agent that does background delivery (unlike serverless hosts
  such as Vercel, which don't fit).
- Everything runs from the terminal (`npm i -g @railway/cli`). The ONLY
  interactive step is `railway login` — it just opens the browser to sign in,
  and that's it; the rest is CLI.
- Then `railway init`, set the runtime secrets as env vars
  (`WALLET_KEYSTORE_JSON`, `WALLET_PASSWORD`, the LLM provider key,
  `AGENT_PORT=${{PORT}}`), and `railway up`. Keep the keystore/secrets in env
  vars only — never baked into the image.
- This is a self-host path: it does NOT use `bag deploy` or AWS, and never
  bundles the keystore into the artifact.

If the user does not bring up deployment, ignore this section and continue the
local flow normally.

## Never do

- Never deploy to AWS / run `bag deploy` / register ERC-8004.
- Never put an LLM in the quote/pricing path or expose signing as a tool.
- Never widen the signing policy or `allowed_hosts` silently.
- Never print or commit the private key or `WALLET_PASSWORD`.
- Never ask the user to confirm a step — proceed autonomously; stop only at the
  funding checkpoint.
