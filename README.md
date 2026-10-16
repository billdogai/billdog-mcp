<div align="center">

# 🐶 BillDog MCP Server

**Set up and inspect paywalls, subscriptions and analytics for your app — from your AI coding agent.**

[![MCP](https://img.shields.io/badge/MCP-Streamable%20HTTP-6E56CF)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/tools-45-2ea44f)](#tools)
[![Auth](https://img.shields.io/badge/auth-OAuth%202.1%20%2B%20PKCE-0969da)](#sign-in-with-billdog)
[![Docs](https://img.shields.io/badge/docs-billdog.io-black)](https://billdog.io/docs/mcp-server)

`https://api.billdog.io/mcp`

[Install](#install) · [What you can ask](#what-you-can-ask) · [Tools](#tools) · [Permissions](#permissions--safety) · [Troubleshooting](#troubleshooting)

</div>

---

[BillDog](https://billdog.io) is paywalls, in-app purchases, subscriptions, analytics, messaging, surveys and journeys behind one SDK for iOS, Android, React Native, Flutter, Expo and the web. This repository is how AI agents connect to it:

- **A hosted MCP server.** 45 tools: diagnose an integration, build the product catalog, design and publish paywalls, and query revenue and retention.
- **A Claude Code plugin.** The server plus two skills: wiring the SDK into an app, and migrating from RevenueCat, Superwall, Adapty and others.

Nothing runs on your machine. The server is hosted by BillDog and every client connects to the same URL.

## Install

You need a free BillDog account at [billdog.io](https://billdog.io) with at least one project.

| Client | How |
|---|---|
| **Claude Code** (plugin, recommended) | `/plugin marketplace add billdogai/billdog-mcp`, then `/plugin install billdog@billdog` |
| **Claude Code** (server only) | `claude mcp add --transport http billdog https://api.billdog.io/mcp` |
| **Claude.ai / Claude Desktop** | Settings → Connectors → **Add custom connector** → paste `https://api.billdog.io/mcp` |
| **Cursor** | Settings → MCP → Add server, or the config below |
| **VS Code (Copilot)** | Command palette → *MCP: Add Server* → HTTP → paste the URL |
| **Antigravity** | Add the server to `mcp_config.json` using the config below |
| **Windsurf / others** | Any client that supports remote Streamable HTTP servers |

The first time a tool runs, your browser opens **Sign in with BillDog**. That's the whole setup.

<details>
<summary><b>Config file snippets</b></summary>

**Cursor** (`~/.cursor/mcp.json`) and most clients:

```json
{
  "mcpServers": {
    "billdog": { "url": "https://api.billdog.io/mcp" }
  }
}
```

**VS Code** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "billdog": { "type": "http", "url": "https://api.billdog.io/mcp" }
  }
}
```

**Antigravity** (`mcp_config.json`, which uses `serverUrl` rather than `url`):

```json
{
  "mcpServers": {
    "billdog": { "serverUrl": "https://api.billdog.io/mcp" }
  }
}
```

**Clients that only launch local (stdio) servers**, bridged with [`mcp-remote`](https://www.npmjs.com/package/mcp-remote):

```json
{
  "mcpServers": {
    "billdog": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://api.billdog.io/mcp"]
    }
  }
}
```

</details>

## Sign in with BillDog

The server uses OAuth 2.1 with PKCE and dynamic client registration, so your client registers and authenticates itself:

1. Your client discovers the authorization server from the server's `401` ([RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)).
2. It registers itself and opens BillDog in your browser.
3. You sign in, pick **one project**, and choose what the agent may do.
4. The agent gets a short-lived token for that project only. It refreshes on its own, and you can disconnect it at any time.

### Using a key instead

For scripts, CI, or clients without OAuth, create a key at **[Dashboard → AI Assistant (MCP) Keys](https://billdog.io/dashboard/analytics/api-keys)**. MCP keys start with `bd_mcp_` and are **read-only**.

```bash
claude mcp add --transport http billdog https://api.billdog.io/mcp \
  --header "Authorization: Bearer bd_mcp_YOUR_KEY"
```

> [!IMPORTANT]
> `bd_public_…` and `bd_secret_…` keys are **SDK keys** for your app and won't work here. The server tells you so if you paste one.

## What you can ask

> *"Why isn't my paywall showing on iOS?"*
> → `get_setup_status` and `resolve_placement_preview` show exactly what's missing, with dashboard links.

> *"Set up monthly and annual Pro plans with a 7-day trial and an `onboarding` placement."*
> → `apply_setup_plan` creates the products, entitlement, offering and placement in one step. Everything starts inactive.

> *"Design a dark paywall for the annual plan from a template, show me a preview, then publish it."*
> → `list_templates` → `create_paywall_from_template` → `update_paywall` → `preview_paywall` → `publish_paywall`

> *"What was MRR last month and which paywall converts best?"*
> → `run_analytics_query`

> *"Migrate this app from RevenueCat."*
> → the `billdog-migration` skill (Claude Code plugin) rewrites the call sites and recreates the catalog.

## Tools

**23 read** · **22 write** · *publish* means it changes what live users see and needs the publish permission.
Machine-readable list: [`tools.json`](tools.json).

### Setup & diagnostics

| Tool | Access | What it does |
|---|---|---|
| `get_setup_status` | read | Inspect the setup status of BillDog for this project. |
| `get_project` | read | Get project metadata including bundle ID, package name, and registered domains. |
| `get_recent_sdk_activity` | read | Inspect recent SDK events and session replays received by BillDog to verify that your app integration is sending data. |
| `resolve_placement_preview` | read | Dry-run preview of what a client SDK receives when resolving a placement. |

### Analytics

| Tool | Access | What it does |
|---|---|---|
| `run_analytics_query` | read | Run a single analytics report. |
| `discover_schema` | read | Return the full BillDog analytics schema — every (source, metric) pairing the project supports, with a 1-line description of what each returns. |
| `list_event_names` | read | List the event names this project received in the last `days` days (default 30), most frequent first, with each one's count and unique users. |
| `list_cohorts` | read | List saved cohorts for this project. |
| `list_saved_views` | read | List saved analytics views for this project. |
| `get_saved_view_results` | read | Re-run a saved view and return its results. |

### Catalog: products, entitlements, offerings, placements

| Tool | Access | What it does |
|---|---|---|
| `list_products` | read | List all products configured in this project's catalog, including their identifiers, product types, subscription durations, and store mapping IDs. |
| `list_entitlements` | read | List all entitlements in this project and the product IDs associated with each entitlement. |
| `list_offerings` | read | List all offerings in this project, showing their packages, attached products, and default/current status. |
| `list_placements` | read | List all placements in this project, including access gating settings and linked offerings. |
| `apply_setup_plan` | write · publish | Declaratively evaluate and execute a complete monetization catalog setup (products, entitlements, offerings, packages, placements). |
| `create_product` | write | Create a new subscription or in-app product in the project catalog. |
| `update_product` | write | Update metadata, store IDs, or display pricing for an existing product in this project. |
| `create_entitlement` | write | Create a new entitlement in the catalog and optionally link it to products. |
| `update_entitlement` | write | Update display name, description, or linked products for an existing entitlement. |
| `set_entitlement_products` | write | Set the explicit list of product IDs granting an entitlement. |
| `create_offering` | write | Create a new offering with packages. |
| `manage_offering_packages` | write | Add or remove packages within an existing offering. |
| `set_current_offering` | write · publish | Promote an offering to be the project current offering. |
| `create_placement` | write | Create a new placement in the project. |
| `link_placement` | write · publish | Link one or more offerings to a placement with priority order. |

### Paywalls

| Tool | Access | What it does |
|---|---|---|
| `list_paywalls` | read | List all paywalls in this project, including active state, wall type, and publication timestamps. |
| `get_paywall` | read | Get details and published layout configuration for a specific paywall in this project by UUID or identifier. |
| `list_templates` | read | List available paywall and onboarding templates from the BillDog template catalog. |
| `create_paywall_from_template` | write | Create a new paywall in the project from an existing template. |
| `design_paywall` | write | Design and compile a paywall from an AI recipe or natural language prompt using the built-in compiler. |
| `update_paywall` | write | Apply constrained patch updates to a draft paywall (copy, title, subtitle, CTA text, brand colors, offering binding). |
| `preview_paywall` | read | Evaluate paywall publish gate (validating components, bindings, offerings) and generate a secure short-lived preview link and token. |
| `publish_paywall` | write · publish | Publish a draft paywall to make it active for SDK delivery. |
| `unpublish_paywall` | write · publish | Deactivate an active paywall. |

### Projects & SDK keys

| Tool | Access | What it does |
|---|---|---|
| `list_projects` | read | List all BillDog projects accessible to the authenticated user, identifying the current active project, bundle ID, and platform. |
| `create_project` | write | Create a new BillDog project for the authenticated user with a specified name, bundle ID, and platform. |
| `create_public_sdk_key` | write | Generate a client-safe public SDK key (bd_public_live_... |

### Engagement: surveys, campaigns, journeys, messages

| Tool | Access | What it does |
|---|---|---|
| `list_surveys` | read | List all multi-page surveys and micro-survey walls configured for this project. |
| `create_survey` | write | Create a survey in the project with custom questions (CSAT, NPS, rating, single-choice) or mobile display settings. |
| `list_campaigns` | read | List all marketing and engagement campaigns configured for this project. |
| `create_campaign` | write | Create a new campaign targeting user segments or in-app placements. |
| `list_journeys` | read | List all customer onboarding and engagement journeys with their sequence of steps. |
| `create_journey` | write | Create a multi-step user journey with triggers and custom actions. |
| `list_message_templates` | read | List all in-app messaging modals/sheets and push notification templates. |
| `create_message_template` | write | Create an in-app message template or push notification template. |

## Permissions & safety

| Guarantee | How |
|---|---|
| **One project per connection** | The project is fixed by your sign-in or key. Project ids in tool arguments are ignored, so the agent can't read another project. |
| **You choose the access** | On the consent screen: read only, read + create/edit, or read + create/edit + publish. Keys are always read-only. |
| **Nothing goes live by accident** | New products, offerings, placements and paywalls start inactive or as drafts. Publishing, setting the current offering and linking a placement with live traffic need the publish permission. |
| **No customer contact data** | Email, phone, display name and custom attributes are withheld from every response, and the response says so. |
| **Audited** | Every write is recorded in the project's audit log, attributed to the agent and the user who connected it. |
| **Revocable** | Disconnect an agent, or revoke a key, from the dashboard at any time. |

## Claude Code plugin

`/plugin install billdog@billdog` gives you the server plus two skills:

- **`billdog-integration`**: adds the SDK to an app (Web, React Native, Expo, iOS, Android) or diagnoses one that's already wired. It covers the identifier traps (`@billdog.io`, not `@billdog`; `io.billdog`, not `com.billdog`), configuration, placements and a verification checklist.
- **`billdog-migration`**: moves an app off RevenueCat, Superwall, Adapty, Mixpanel, Amplitude, OneSignal, Braze, Typeform or SurveyMonkey. It covers both the code and the data.

## Troubleshooting

| Message | Fix |
|---|---|
| *"This is a BillDog SDK key…"* | You pasted a `bd_public_`/`bd_secret_` key. Use Sign in with BillDog, or create a `bd_mcp_` key. |
| *"This key has MCP access turned off"* | Turn MCP access on for the key on the [MCP keys page](https://billdog.io/dashboard/analytics/api-keys). |
| *…lacks the `mcp:write` scope* | Keys are read-only. Connect with Sign in with BillDog and allow **Create and edit configuration** (or **Publish and go live** for `mcp:publish`). |
| Browser didn't open | Run `/mcp` in Claude Code and choose **Authenticate**, or reconnect the connector. |
| *"Unknown or revoked client_id"* | Remove the BillDog server from your client and add it again. |

## Repository layout

One plugin at the repository root, in two formats so every client can read it:

```
plugin.json                       Agent Plugins / Open Plugins manifest (Cursor, cursor.directory, other clients)
mcp.json                          Agent Plugins MCP config (streamable-http)
skills/                           billdog-integration, billdog-migration (shared by both formats)
.claude-plugin/marketplace.json   Claude Code marketplace (this repo)
.claude-plugin/plugin.json        Claude Code plugin manifest
.mcp.json                         Claude Code MCP config
server.json                       official MCP Registry entry
smithery.yaml                     Smithery (stdio bridge via mcp-remote)
glama.json                        Glama ownership claim
tools.json                        all 45 tools, grouped, with access level
assets/icon.png                   listing icon
```

## Links

- 📖 Docs: [billdog.io/docs/mcp-server](https://billdog.io/docs/mcp-server)
- 🧭 Full SDK reference for agents: [billdog.io/SKILL.md](https://billdog.io/SKILL.md)
- 🐛 Issues: [github.com/billdogai/billdog-mcp/issues](https://github.com/billdogai/billdog-mcp/issues)
