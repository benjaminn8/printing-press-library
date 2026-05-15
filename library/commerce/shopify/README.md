# Shopify CLI

**Operate a Shopify store from the terminal with curated Admin GraphQL commands, local sync, analytics, and bulk exports.**

Endpoint mirrors for orders, products, customers, inventory items, and fulfillment orders. A local SQLite store for offline reads and full-text search after sync. MCP server with both stdio and HTTP transport so agents (OpenAI Codex CLI, hosted clients) consume the same surface without learning GraphQL.

Learn more at [Shopify](https://shopify.dev/docs/api/admin-graphql).

## Install

The recommended path installs both the `shopify-pp-cli` binary and the `pp-shopify` agent skill in one shot:

```bash
npx -y @mvanhorn/printing-press install shopify
```

For CLI only (no skill):

```bash
npx -y @mvanhorn/printing-press install shopify --cli-only
```


### Without Node (Go fallback)

If `npx` isn't available (no Node, offline), install the CLI directly via Go (requires Go 1.26.3 or newer):

```bash
go install github.com/mvanhorn/printing-press-library/library/commerce/shopify/cmd/shopify-pp-cli@latest
```

This installs the CLI only — no skill.

### Pre-built binary

Download a pre-built binary for your platform from the [latest release](https://github.com/mvanhorn/printing-press-library/releases/tag/shopify-current). On macOS, clear the Gatekeeper quarantine: `xattr -d com.apple.quarantine <binary>`. On Unix, mark it executable: `chmod +x <binary>`.

<!-- pp-hermes-install-anchor -->
## Install for Hermes

From the Hermes CLI:

```bash
hermes skills install mvanhorn/printing-press-library/cli-skills/pp-shopify --force
```

Inside a Hermes chat session:

```bash
/skills install mvanhorn/printing-press-library/cli-skills/pp-shopify --force
```

## Install for OpenClaw

Tell your OpenClaw agent (copy this):

```
Install the pp-shopify skill from https://github.com/mvanhorn/printing-press-library/tree/main/cli-skills/pp-shopify. The skill defines how its required CLI can be installed.
```

## Authentication

Set SHOPIFY_ACCESS_TOKEN to a custom-app token with the read scopes you need (read_orders, read_products, read_customers, read_inventory, read_fulfillments).

## Quick Start

```bash
# Confirm auth and shop reachability.
shopify-pp-cli doctor


# Local archive state for all resources before any sync.
shopify-pp-cli workflow status --json


# List recent orders with agent-native JSON.
shopify-pp-cli orders list --first 10 --json


# Cross-table FTS over synced data.
shopify-pp-cli search 'pending' --json


# Inspect the current Shopify bulk export job.
shopify-pp-cli bulk-operations current --json

```

## Usage

Run `shopify-pp-cli --help` for the full command reference and flag list.

## Commands

### abandoned-checkouts

Shopify abandoned checkouts for recovery campaigns and lost-cart analysis.

- **`shopify-pp-cli abandoned-checkouts get`** - Get one Shopify abandoned checkout by GraphQL ID.
- **`shopify-pp-cli abandoned-checkouts list`** - List abandoned checkouts from the Shopify Admin GraphQL API.

### customers

Shopify customers with lifetime order count, lifetime spend, and contact fields.

- **`shopify-pp-cli customers get`** - Get one Shopify customer by GraphQL ID.
- **`shopify-pp-cli customers list`** - List customers from the Shopify Admin GraphQL API.

### fulfillment-orders

Shopify fulfillment orders for lag, routing, and fulfillment-state analysis.

- **`shopify-pp-cli fulfillment-orders get`** - Get one Shopify fulfillment order by GraphQL ID.
- **`shopify-pp-cli fulfillment-orders list`** - List fulfillment orders from the Shopify Admin GraphQL API.

### inventory-items

Shopify inventory items with tracked status and available quantities by location.

- **`shopify-pp-cli inventory-items get`** - Get one Shopify inventory item by GraphQL ID.
- **`shopify-pp-cli inventory-items list`** - List inventory items from the Shopify Admin GraphQL API.

### orders

Shopify orders with money totals, financial state, and fulfillment state.

- **`shopify-pp-cli orders get`** - Get one Shopify order by GraphQL ID.
- **`shopify-pp-cli orders list`** - List orders from the Shopify Admin GraphQL API.

### products

Shopify products with product status, catalog metadata, and a compact variant inventory projection.

- **`shopify-pp-cli products get`** - Get one Shopify product by GraphQL ID.
- **`shopify-pp-cli products list`** - List products from the Shopify Admin GraphQL API.


## Output Formats

```bash
# Human-readable table (default in terminal, JSON when piped)
shopify-pp-cli abandoned-checkouts list

# JSON for scripting and agents
shopify-pp-cli abandoned-checkouts list --json

# Filter to specific fields
shopify-pp-cli abandoned-checkouts list --json --select id,name,status

# Dry run — show the request without sending
shopify-pp-cli abandoned-checkouts list --dry-run

# Agent mode — JSON + compact + no prompts in one flag
shopify-pp-cli abandoned-checkouts list --agent
```

## Agent Usage

This CLI is designed for AI agent consumption:

- **Non-interactive** - never prompts, every input is a flag
- **Pipeable** - `--json` output to stdout, errors to stderr
- **Filterable** - `--select id,name` returns only fields you need
- **Previewable** - `--dry-run` shows the request without sending
- **Read-only by default** - this CLI does not create, update, delete, publish, send, or mutate remote resources
- **Offline-friendly** - sync/search commands can use the local SQLite store when available
- **Agent-safe by default** - no colors or formatting unless `--human-friendly` is set

Exit codes: `0` success, `2` usage error, `3` not found, `4` auth error, `5` API error, `7` rate limited, `10` config error.

## Freshness

This CLI owns bounded freshness for registered store-backed read command paths. In `--data-source auto` mode, covered commands check the local SQLite store before serving results; stale or missing resources trigger a bounded refresh, and refresh failures fall back to the existing local data with a warning. `--data-source local` never refreshes, and `--data-source live` reads the API without mutating the local store.

Set `SHOPIFY_NO_AUTO_REFRESH=1` to disable the pre-read freshness hook while preserving the selected data source.

Covered command paths:
- `shopify-pp-cli abandoned-checkouts`
- `shopify-pp-cli abandoned-checkouts get`
- `shopify-pp-cli abandoned-checkouts list`
- `shopify-pp-cli customers`
- `shopify-pp-cli customers get`
- `shopify-pp-cli customers list`
- `shopify-pp-cli fulfillment-orders`
- `shopify-pp-cli fulfillment-orders get`
- `shopify-pp-cli fulfillment-orders list`
- `shopify-pp-cli inventory-items`
- `shopify-pp-cli inventory-items get`
- `shopify-pp-cli inventory-items list`
- `shopify-pp-cli orders`
- `shopify-pp-cli orders get`
- `shopify-pp-cli orders list`
- `shopify-pp-cli products`
- `shopify-pp-cli products get`
- `shopify-pp-cli products list`

JSON outputs that use the generated provenance envelope include freshness metadata at `meta.freshness`. This metadata describes the freshness decision for the covered command path; it does not claim full historical backfill or API-specific enrichment.

## Runtime Endpoint

This CLI resolves endpoint placeholders at runtime, so one installed binary can target different tenants or API versions without regeneration.

Endpoint environment variables:
- `SHOPIFY_SHOP` resolves `{shop}`
- `SHOPIFY_API_VERSION` resolves `{api_version}`

Base URL: `https://{shop}`

GraphQL path: `/admin/api/{api_version}/graphql.json`

## Use with Claude Code

Install the focused skill — it auto-installs the CLI on first invocation:

```bash
npx skills add mvanhorn/printing-press-library/cli-skills/pp-shopify -g
```

Then invoke `/pp-shopify <query>` in Claude Code. The skill is the most efficient path — Claude Code drives the CLI directly without an MCP server in the middle.

<details>
<summary>Use as an MCP server in Claude Code (advanced)</summary>

If you'd rather register this CLI as an MCP server in Claude Code, install the MCP binary first:


```bash
go install github.com/mvanhorn/printing-press-library/library/commerce/shopify/cmd/shopify-pp-mcp@latest
```

Then register it:

```bash
claude mcp add shopify shopify-pp-mcp -e SHOPIFY_SHOP=<your-store>.myshopify.com -e SHOPIFY_API_VERSION=2026-04 -e SHOPIFY_ACCESS_TOKEN=<your-key>
```

</details>

## Use with Claude Desktop

This CLI ships an [MCPB](https://github.com/modelcontextprotocol/mcpb) bundle — Claude Desktop's standard format for one-click MCP extension installs (no JSON config required).

To install:

1. Download the `.mcpb` for your platform from the [latest release](https://github.com/mvanhorn/printing-press-library/releases/tag/shopify-current).
2. Double-click the `.mcpb` file. Claude Desktop opens and walks you through the install.
3. Fill in `SHOPIFY_ACCESS_TOKEN` when Claude Desktop prompts you.

Requires Claude Desktop 1.0.0 or later. Pre-built bundles ship for macOS Apple Silicon (`darwin-arm64`) and Windows (`amd64`, `arm64`); for other platforms, use the manual config below.

<details>
<summary>Manual JSON config (advanced)</summary>

If you can't use the MCPB bundle (older Claude Desktop, unsupported platform), install the MCP binary and configure it manually.


```bash
go install github.com/mvanhorn/printing-press-library/library/commerce/shopify/cmd/shopify-pp-mcp@latest
```

Add to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "shopify": {
      "command": "shopify-pp-mcp",
      "env": {
        "SHOPIFY_SHOP": "<your-store>.myshopify.com",
        "SHOPIFY_API_VERSION": "2026-04",
        "SHOPIFY_ACCESS_TOKEN": "<your-key>"
      }
    }
  }
}
```

</details>

## Health Check

```bash
shopify-pp-cli doctor
```

Verifies configuration, credentials, and connectivity to the API.

## Configuration

Config file: `~/.config/shopify-pp-cli/config.toml`

Static request headers can be configured under `headers`; per-command header overrides take precedence.

Environment variables:

| Name | Kind | Required | Description |
| --- | --- | --- | --- |
| `SHOPIFY_SHOP` | endpoint | Yes |  |
| `SHOPIFY_API_VERSION` | endpoint | Yes |  |
| `SHOPIFY_ACCESS_TOKEN` | per_call | Yes | Set to your API credential. |

## Troubleshooting
**Authentication errors (exit code 4)**
- Run `shopify-pp-cli doctor` to check credentials
- Verify the environment variable is set: `echo $SHOPIFY_ACCESS_TOKEN`
**Not found errors (exit code 3)**
- Check the resource ID is correct
- Run the `list` command to see available items

### API-specific

- **401 Unauthorized on every call** — Verify SHOPIFY_ACCESS_TOKEN is an Admin API token, not a Storefront API token. Re-issue under custom-app settings.
- **Empty results from local search or analytics** — Run shopify-pp-cli sync --full first; analytics and search read from the local SQLite store.
- **Stale data in queries** — Run shopify-pp-cli workflow status --json to see per-resource state, then sync the specific resource that is stale.

---

Generated by [CLI Printing Press](https://github.com/mvanhorn/cli-printing-press)
