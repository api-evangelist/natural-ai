---
name: natural
description: Send payments, request payments, check balances, move funds, and manage payment agents on Natural.
homepage: https://natural.com
compatibility: Hosted MCP works with HTTP-capable MCP clients.
metadata:
  author: Natural
  version: "2.0"
---

# Natural

Natural is the agentic payments platform. Agents and apps use Natural to hold funds, send and request payments, move money between wallets and linked banks, and operate delegated payment agents for customers.

You are the AI host operating Natural for a user. This is the full skill: setup, the tool reference, the core rules, and the common journeys. If Natural's tools are already available to you (you can see `get_account_balance` and friends), skip setup, make one read-only call to confirm, and go to the journey you need.

## MCP Tools

The hosted MCP server exposes intent-shaped Natural tools:

| Tool                                           | Purpose                                                              |
| ---------------------------------------------- | -------------------------------------------------------------------- |
| `get_transaction_status`                       | Look up one payment or transfer by `pay_*` or `trf_*` ID             |
| `wait_for_transaction`                         | Block until a payment or transfer reaches a terminal status          |
| `get_payment_request`                          | Look up one payment request by `prq_*` ID                            |
| `list_transactions`                            | List paginated transaction history                                   |
| `get_account_balance`                          | Read wallet balances and holds                                       |
| `get_party_limits`                             | Read the party's per-transaction, daily, and monthly spend limits    |
| `list_wallets`                                 | List wallets with id, name, type, status, default flag, and balance  |
| `get_identity`                                 | Read the caller party, acting agent, handles, and permissions        |
| `list_external_accounts`                       | List linked external accounts and connection status                  |
| `get_external_account`                         | Look up one linked external account by `eac_*` ID                    |
| `create_external_account_from_processor_token` | Link or refresh an external account from a Plaid processor token     |
| `create_payment`                               | Send money to an email, phone, `pty_*`, or `agt_*` recipient         |
| `cancel_payment`                               | Cancel a pending-claim outbound payment before it is claimed         |
| `request_payment`                              | Request money from an email, phone, `pty_*`, or `agt_*` payer        |
| `fulfill_payment_request`                      | Pay an incoming payment request (`prq_*`) from your wallet or bank   |
| `decline_payment_request`                      | Decline an incoming payment request (`prq_*`) without paying it      |
| `deposit_funds`                                | Pull funds from a linked bank account or return funding instructions |
| `withdraw_funds`                               | Push funds to a linked bank account                                  |
| `transfer_between_wallets`                     | Move funds between two wallets of the authenticated party            |
| `list_agents`                                  | List Natural agents for the authenticated party                      |
| `list_customers`                               | List customer relationships and delegation status                    |
| `create_agent`                                 | Create a new Natural agent for the authenticated party               |
| `invite_customer`                              | Invite a customer to delegate authority to an agent                  |
| `get_funding_options`                          | List linked accounts and ACH push-to-wallet instructions             |

With an agent-scoped OAuth grant or agent key, the credential already carries agent identity. Money-moving tools still require `instanceId`. With a user-scoped grant or party API key, calls act as the user or party by default; existing integrations may still pass `agentId` for attribution, though agent-scoped OAuth or agent keys are preferred for new agent integrations.

## Core rules (always apply)

- **MCP money is explicit and exact.** Pass a canonical decimal string at the currency's exact precision plus currency, for example `amount: "5.38", currency: "USD"`. Preserve the user's numeric value: `$5` becomes `"5.00"` and `$5.3` becomes `"5.30"`. If the value cannot be represented exactly at that currency's precision (for example USD `$1.983475`), do not call a money-moving tool. Ask the user which exact amount to send; never choose a rounded or truncated value for them. Never send bare integer strings, integer cents, JSON numbers, currency symbols, commas, excess precision, or guessed values.
- **Read money as one value.** Results use objects such as `amount: { decimal: "5.00", currency: "USD", display: "USD 5.00" }`. Use `display` when talking to a person and copy `decimal` plus `currency` when another tool requires the same value.
- **Confirm before money moves.** For `create_payment`, `request_payment`, `fulfill_payment_request`, `decline_payment_request`, `cancel_payment`, `deposit_funds`, `withdraw_funds`, and `transfer_between_wallets`: state the amount, currency, recipient or payer, and what you are about to do, then wait for an explicit yes.
- **Do not claim Natural is connected until a read-only call actually succeeds** (`get_account_balance` or `list_agents`). Config written is not the same as tools callable.
- **Idempotency keys must be deterministic per business event,** not a fresh UUID per retry. Derive from a stable id (for example `payout-{invoice.id}`). A new key on retry creates a duplicate payment.
- **KYB is a hard gate.** Payments cannot move until business verification completes. It is human-reviewed, so plan for hours to days, not minutes.
- **A 404 means the resource does not exist or you lack access.** Do not probe to tell the two apart.
- **Use MCP tools** (or the CLI fallback below). Never write raw HTTP, curl, or requests against `api.natural.com`. Never spawn a subprocess or child agent to load the tools; reload this session instead.
- **Do not log** full account numbers, SSNs, customer emails, or full party IDs.

## Talking to the user

Be plainly honest about what you are doing with their money:

- Say what you are doing in plain terms: "Connecting Natural so I can run this," or "I'll verify the balance, then send the request."
- Before anything moves, show the exact amount, recipient or payer, and that you will ask first.
- If something is not available here (a geography, card acquiring, a capability the account does not expose), say so and offer the supported Natural path. Do not overstate what Natural can do, and do not hide a step that affects the user's money.

## Setup

Default to the hosted MCP server at `https://mcp.natural.com`. It works across desktop, mobile, web, and agent hosts.

How you connect depends on what kind of host you are:

- **You can run commands or edit config** (Claude Code, Codex, Cursor, a terminal, an agent framework): add the MCP server yourself with the _Generic flow_ and _Per-host specifics_ below.
- **You are a chat app** (ChatGPT, or Claude on web, desktop, or mobile): you do not install anything. You add Natural the way these apps add any tool, by turning on a connector in settings, which the user does once. See _Chat apps_ next, then continue normally.

### Chat apps (ChatGPT, Claude on web, desktop, mobile)

Connecting Natural here is quick and routine: the user turns on a connector once, signs in, and Natural's tools appear right in this conversation. You already have everything you need from this chat, so guide the user in plain language and stay with them until it is done.

Custom connectors are a paid feature in both ChatGPT and Claude, so first ask the user which plan they are on, then take the matching path:

- **Paid plan** (ChatGPT Plus, Pro, Business, or Enterprise; Claude Pro, Max, Team, or Enterprise): add Natural as a connector with the steps for their app below.
- **Free plan:** connectors are not available there, so use Natural in the dashboard at `natural.com`, which does everything without a connector. Offer to walk them through it.

**ChatGPT** (connectors live under Developer mode):

1. Open **Settings -> Apps & Connectors**, scroll to **Advanced settings**, and turn on **Developer mode**.
2. Back on **Apps & Connectors**, click **Create**. Name it `Natural` and enter the URL `https://mcp.natural.com/mcp`.
3. Click **Create**, then sign in on Natural's page when it opens and approve the access it requests.
4. Open **Natural** in the connector list and make sure its tools are switched on.

**Claude (web, desktop, or mobile):**

1. Open **Settings -> Connectors**.
2. Choose **Add custom connector** and enter `https://mcp.natural.com/mcp`.
3. Click **Connect** and approve Natural's authorization page.

When the connector is on and signed in, Natural's tools are available here. Make one read-only call (`get_account_balance` or `list_agents`) to confirm, then pick up exactly where the user's request left off. If the tools are not there yet, have the user send a new message or open a new chat so the app loads the connector, then verify and continue.

### Generic flow (most hosts)

1. Add the remote MCP server `natural` at `https://mcp.natural.com` (no `Authorization` header). Prefer the host's own discovery-first add command (one that takes an OAuth flag) over hand-writing config; a bare URL entry defaults to no-auth on many hosts and silently skips sign-in.
2. Start the host's MCP OAuth / login flow. The browser opens Natural's authorization page.
3. Approve the four connector scopes: `payments:move` (view/send/request/cancel payments, deposits, withdrawals, balances), `external-accounts:link` (link bank accounts using processor tokens), `agents:operate` (list customers, create agents), `customers:invite` (send/revoke delegation invites). Do not request `payments:view`, `payments:send`, or `payments:full`; those add duplicate rows.
4. The consent screen asks what to act as: As me, New agent, or an existing agent. New agent provisions the agent during approval; reconnecting the same agent should pick the existing agent. Because the agent is provisioned at consent, do not call `create_agent` as a setup follow-up unless the user explicitly wants a separate one.
5. **Verify before claiming connected:** make one read-only call (`get_account_balance` or `list_agents`). A config entry, `mcp list` (reads config), or an MCP resources view are not proof. The proof is a successful tool call.

### Per-host specifics

| Host          | Install                                                                                                                                               | Auth                                                                                                      | Notes                                                                                                                                                                                                                                                                                                     |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Claude Code   | `/plugin marketplace add naturalpay/agent-plugins` then `/plugin install natural@natural`, `/reload-plugins`, `/mcp`                                  | In `/mcp` pick **natural**, choose **Authenticate**                                                       | Plugin reloads in-session via `/reload-plugins`. Also connect the **natural** connector in Claude desktop/claude.ai under Settings -> Connectors (separate step). Manual fallback: `claude mcp add --transport http natural https://mcp.natural.com --scope user` (no in-session reload, restart needed). |
| Codex         | shell: `codex plugin marketplace add naturalpay/agent-plugins`; then `/plugins` or `codex plugin add natural@natural`; then `codex mcp login natural` | `codex mcp login natural [--scopes payments:move,external-accounts:link,agents:operate,customers:invite]` | Three separate phases. `/mcp` shows tools, but if you still cannot call `mcp__natural.*`, restart and resume with `codex resume <id>`. Manual fallback: `codex mcp add natural --url https://mcp.natural.com`.                                                                                            |
| Cursor        | merge `{"mcpServers":{"natural":{"url":"https://mcp.natural.com"}}}` into `~/.cursor/mcp.json`                                                        | `agent mcp login natural`                                                                                 | Verify `agent mcp list-tools natural`. Refresh MCP tools or restart the window if not shown.                                                                                                                                                                                                              |
| Hermes (Nous) | `hermes mcp add natural --url https://mcp.natural.com --auth oauth`                                                                                   | `hermes mcp login natural`                                                                                | The `--auth oauth` flag is required; without it Hermes defaults to `auth: none`, sends an unauthenticated request, gets a correct `401`, and never starts OAuth. Confirm `hermes mcp test natural` reports `Auth: OAuth 2.1 PKCE`.                                                                        |
| Any other     | probe the host for an MCP add/configure command and an OAuth flag; add `https://mcp.natural.com` with the auth mode set, not just the URL             | host's MCP login flow                                                                                     | Natural always advertises OAuth (RFC 9728 metadata + `401` with `WWW-Authenticate`); `auth: none` means the host config is missing the auth mode, not that Natural lacks OAuth.                                                                                                                           |

### If the tools do not show after install (reload steps)

Adding the server does not always make the tools callable in the running session. This is a reload, not a failure, and the user should not have to retype their request. Do the steps for your host, in order.

**Claude Code**

1. Run `/reload-plugins`, then `/mcp`. The plugin usually loads in-session.
2. If `/mcp` does not show **natural** as connected, exit the session (`/exit`, or press Ctrl-C twice).
3. Resume in the same terminal with `claude -c`. Your original prompt and plan come back automatically; do not retype them.
4. Run `/mcp`, pick **natural**, choose **Authenticate** if it is not authorized yet.
5. Verify with a read-only call (below) before doing anything else.

**Codex**

Codex does not hot-load a plugin install, so expect the running session to still lack the tools even after `codex mcp login natural` succeeds. `/mcp` listing **natural** is not proof you can call it.

1. Exit the session: press Ctrl-C twice, or type `/exit`. On exit, Codex prints a resume command.
2. Run that resume command (`codex resume <id>`) in the same terminal. Your prompt and plan are restored; do not retype them.
3. Verify with a read-only call (below) before doing anything else.

**Cursor**: refresh MCP tools, or restart the window, then verify with a read-only call.

**Manual `mcp add` (any host)**: a manually added, non-plugin server has no in-session reload, so a restart is the only path. Exit, restart the host, resume the session if it supports it, then verify.

After any reload or resume, every time:

- **Verify first:** make one read-only call (`get_account_balance` or `list_agents`). Only a successful call proves the tools are callable.
- **Auth or insufficient-scope error?** Re-run the host's login (re-authorize), do not leave MCP.
- **Never auto-execute a payment on resume,** even if the original request named the amount and recipient. Confirm with the user first.
- **Never spawn a subprocess or child agent to "load the tools."** A non-interactive child cannot show the money-movement approval prompt, so payments are auto-cancelled and then misreported as "user cancelled" when the user never saw a prompt. Reload this session instead.

### CLI (structural fallback only)

Use the CLI only when a host architecturally cannot speak MCP at all (no remote-MCP transport, subprocess-only Claude invocation; OpenClaw is the canonical case). First exhaust install + reload/restart + re-auth + config-check + read-only verify. Then:

```bash
curl -fsSL https://natural.com/install.sh | bash
natural login
natural <resource> <command> --format json
```

Same browser OAuth, no API key. Never an API key, never raw HTTP.

## Journeys

### Send money

1. Confirm with the user the recipient, the amount in dollars, and the payment description.
2. Express it as an exact decimal string with currency, for example `amount: "50.00", currency: "USD"`. Do not convert it to cents.
3. Call `create_payment` with `amount`, `currency`, `recipient` (email, phone, `pty_*`, or `agt_*`), `description` (required, at least 20 characters), a deterministic `idempotencyKey`, and `agentId`+`instanceId` when available. Response includes a `pay_*` id; when status is `PENDING_CLAIM`, Natural delivers the claim link to the recipient directly; it never appears in the tool result. Report `PENDING_CLAIM` to the user and do not wait on it: the recipient claims on their own time, which can take days. For other in-flight statuses, confirm the outcome with `wait_for_transaction` (same `pay_*` id; blocks until a terminal status or returns the latest status with `timedOut: true`); use `get_transaction_status` for one-off snapshots.

### Request money from someone

1. Confirm the payer, amount in dollars, and description.
2. Express it as an exact decimal string with currency, for example `amount: "5.00", currency: "USD"`.
3. Call `request_payment` with `payer`, `amount`, `currency`, `description` (optional), and a deterministic `idempotencyKey`. Response includes a `prq_*` id; give it to the user to reference. The payer fulfills or declines on their side.

### Approve or decline a request someone sent you

You need the request's `prq_` id first. **There is no tool that lists incoming payment requests,** and `list_transactions` returns only payments (`pay_*`) and transfers (`trf_*`), never requests (`prq_*`). So you cannot discover or search for a pending request from the tools.

- If the user says "approve the request" without an id, do **not** guess and do **not** infer a request from `list_transactions` (a row there is a payment/transfer with its own direction, not the incoming request). Ask the requester to share the `prq_...` id (from their Natural confirmation or the dashboard).
- With the id: call `get_payment_request` and confirm its `amount.display` and payer with the user. To pay, call `fulfill_payment_request` with `expectedAmount` copied exactly from `amount.decimal`, `currency` copied from `amount.currency`, and a deterministic `idempotencyKey`; Natural re-reads the request and rejects any mismatch before submitting payment. To decline, call `decline_payment_request`. Both are user-facing money decisions: confirm first.

### Check balance, funding, or status

`get_account_balance` for wallet balances and holds, `get_funding_options` for linked accounts and ACH instructions, `get_transaction_status` for a `pay_*`/`trf_*`, `list_transactions` for history.

## Safe first actions

When the user says "try it" without a specific instruction, start read-only: verify the connection, then `list_agents`, `list_customers`, `get_account_balance`, `get_funding_options`, or `list_transactions`. Do not call a money-moving tool until the user supplies the details and confirms the side effect.
