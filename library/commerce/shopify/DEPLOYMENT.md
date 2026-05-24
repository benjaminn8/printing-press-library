# Shopify PP CLI — Deployment Notes

Companion to `ABSORB-MANIFEST.md`. Tracks where the patched binary is deployed, how it got there, and how to rebuild.

## State of this repo

This is the fork `benjaminn8/printing-press-library` (parent: `mvanhorn/printing-press-library`). It carries the 5-bug patch set applied after a v4.6.1 factory reprint on 2026-05-16. Source of truth for any Shopify PP CLI binary running in this org.

Latest patch commit: `580ec6e` (`docs(shopify): add bug #5 (--query dead-flag) to ABSORB-MANIFEST`).

## Patches (full reference)

See `ABSORB-MANIFEST.md` for the canonical list. Summary:

| # | Bug | Fix location |
|---|---|---|
| 1 | `shopifyql funnel` hardcoded hallucinated columns | `internal/cli/shopifyql.go` |
| 2 | `--since` formatted local TZ, not UTC | `internal/cli/sync.go` (`.UTC()` before `.Format`) |
| 3 | Default `--max-pages 10` × 50 capped output at 500 | `internal/cli/sync.go` (default 0) |
| 4 | `--query` flag dead on 5 list commands | `internal/cli/{orders,customers,products,inventory_items,abandoned_checkouts}_list.go`. fulfillment-orders left dead (GraphQL field has no `query` arg) |
| 5 | fulfillment-orders sync access denied | Not code — token scope. Grant `read_merchant_managed_fulfillment_orders` to the custom app |

Bug #1 is library-side and belongs in any PR to upstream. Bugs #2, #3, #5 are generator bugs in `mvanhorn/cli-printing-press` (factory) and should PR there separately.

## Known deployments

| Host | Path | Arch | Built | Source |
|---|---|---|---|---|
| iMac (Hermes-Mimi profile, dev) | `~/go/bin/shopify-pp-cli` | darwin/arm64 | 2026-05-16 | local "devel" tree, now deleted |
| hosted_hermes (Atlas client VPS bundle) | `~/claudecode_demo/hosted_hermes/infra/bin/shopify-pp-cli-linux-amd64` | linux/amd64 | 2026-05-20 | this fork at commit `580ec6e` |

**Both deployments ship binary only.** No source on those hosts. The Go module cache entry at `~/go/pkg/mod/github.com/mvanhorn/printing-press-library` on the iMac is pristine upstream — NOT the source of any deployed binary. Build info on the iMac binary shows `mod shopify-pp-cli (devel)`, no VCS revision, confirming it was built from a now-deleted local tree.

This repo (the fork clone at `~/claudecode_demo/printing-press-library` on the Air) is the only surviving working tree.

## Rebuilding

From this repo, on a Go 1.26+ host:

```bash
cd library/commerce/shopify

# Darwin/arm64 (iMac, Air)
CGO_ENABLED=1 go install ./cmd/shopify-pp-cli
# -> $GOBIN/shopify-pp-cli (defaults to ~/go/bin/)

# Linux/amd64 (Atlas client VPS, hosted_hermes bundle)
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags='-s -w' \
  -o shopify-pp-cli-linux-amd64 ./cmd/shopify-pp-cli
# -> copy into hosted_hermes/infra/bin/
```

CGO required for the SQLite driver. Cross-compiling linux from darwin needs a cross toolchain (`brew install FiloSottile/musl-cross/musl-cross` or build on a Linux VM/container).

## Runtime requirements (any host)

- `SHOPIFY_ACCESS_TOKEN` (Admin API access token, ~24h TTL via client_credentials)
- `SHOPIFY_SHOP` (`<store>.myshopify.com`) — note hosted_hermes/Hermes uses `SHOPIFY_STORE_DOMAIN`; the iMac `.env` aliases it to `SHOPIFY_SHOP` for the CLI
- Writable `~/.local/share/shopify-pp-cli/` for SQLite store
- Token refresh job. Reference impl: `~/.hermes/profiles/mimi/scripts/shopify-refresh-token.sh` + launchd `com.benhuang.shopify-refresh` (23h). Each Atlas client VPS needs an equivalent

## Upstream PR plan

Before pushing to `mvanhorn/printing-press-library`, run the pre-PR scrub from `~/claudecode_demo/printing-press/HANDOFF.md`:

1. Strip `benjamin-huang` from copyright headers
2. Fix goreleaser homepage and owner
3. Normalize `.printing-press.json` (spec_path, owner)
4. Commit and push, then open PR

Bug #1 (library-side) goes in the upstream library PR. Bugs #2/#3/#5 are factory-side — separate PRs against `mvanhorn/cli-printing-press`, using the patches applied here as the spec.

## Cross-reference

- Reprint handoff (full bug history, scrub script): `~/claudecode_demo/printing-press/HANDOFF.md`
- Factory (generator) repo: `~/claudecode_demo/printing-press-factory` → `mvanhorn/cli-printing-press`
- hosted_hermes binary README: `~/claudecode_demo/hosted_hermes/infra/bin/README.md`
- Atlas integration shelling out to the binary: `~/claudecode_demo/hosted_hermes/atlas/integrations/shopify.py`
