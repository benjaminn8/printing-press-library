# Shopify PP CLI — Deployment Notes

Companion to `ABSORB-MANIFEST.md`. Tracks where the patched binary is deployed, how it got there, and how to rebuild.

## State of this repo

This is the fork `benjaminn8/printing-press-library` (parent: `mvanhorn/printing-press-library`). It carries the 5-bug patch set applied after a v4.6.1 factory reprint on 2026-05-16. Source of truth for any Shopify PP CLI binary running in this org.

Latest commit: `0d45795d` (`docs(shopify): add DEPLOYMENT.md tracking fork-deployed binaries`, 2026-05-24). Latest code patch: `580ec6e`.

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

| Host | Path | Arch | Built | Source | VCS rev |
|---|---|---|---|---|---|
| Air | `~/.local/bin/shopify-pp-cli` → `~/go/bin/shopify-pp-cli` | darwin/arm64 | 2026-05-24 | fork clone at `~/claudecode_demo/printing-press-library` | `0d45795d` |
| iMac (Hermes-Mimi profile) | `~/.local/bin/shopify-pp-cli` → `~/go/bin/shopify-pp-cli` | darwin/arm64 | 2026-05-24 | fork clone at `~/code/printing-press-library` | `0d45795d` |
| hosted_hermes (Atlas client VPS bundle) | `~/claudecode_demo/hosted_hermes/infra/bin/shopify-pp-cli-linux-amd64` | linux/amd64 | 2026-05-20 | fork at `580ec6e` (pre-DEPLOYMENT.md) | `580ec6e` |

**Air and iMac convention:** `~/.local/bin/shopify-pp-cli` is a symlink to `~/go/bin/shopify-pp-cli`. Both machines have `~/.local/bin` on `$PATH` (not `~/go/bin`), so the symlink is what makes `go install` updates take effect without manual copying.

**Verifying which build is live:** `go version -m ~/go/bin/shopify-pp-cli | grep vcs.revision` prints the source commit. `vcs.modified=false` means it was built from clean committed source.

**Version string is always "1.0.0" on local builds.** That's a hardcoded fallback in `internal/cli/root.go`. Real versions only get injected by goreleaser on official release builds. `--version` output is not a reliable identity check; use `vcs.revision` instead.

**hosted_hermes still ships binary only.** No source on that host. The Linux binary at `~/claudecode_demo/hosted_hermes/infra/bin/shopify-pp-cli-linux-amd64` is pinned at commit `580ec6e` (before DEPLOYMENT.md was added). Rebuild via the cross-compile step in the "Rebuilding" section when patches change.

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

### First-time setup on a fresh darwin host

```bash
git clone https://github.com/benjaminn8/printing-press-library.git ~/code/printing-press-library
cd ~/code/printing-press-library/library/commerce/shopify
go install ./cmd/shopify-pp-cli
ln -sf ~/go/bin/shopify-pp-cli ~/.local/bin/shopify-pp-cli   # only if ~/.local/bin is on $PATH but ~/go/bin is not
shopify-pp-cli --version
go version -m $(which shopify-pp-cli) | grep vcs.revision    # should match origin/main
```

Clone path is convention only: Air uses `~/claudecode_demo/printing-press-library`, iMac uses `~/code/printing-press-library`. Match the existing host or pick anything.

### Update workflow (after a fork patch)

On either machine, after pushing a fix to `origin/main`:

```bash
cd <fork clone>/library/commerce/shopify
git -C ../../.. pull
go install ./cmd/shopify-pp-cli
```

The `~/.local/bin/shopify-pp-cli` symlink picks up the rebuild automatically. No need to touch the symlink again after first setup.

To push the update to the iMac from the Air in one line:

```bash
ssh imac 'cd ~/code/printing-press-library && git pull && cd library/commerce/shopify && go install ./cmd/shopify-pp-cli'
```

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
