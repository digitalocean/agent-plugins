# DOCR MCP tool reference

Source of truth: `pkg/registry/docr/` in
[digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean).
If this table and the live `retrieve_schema` output differ, trust the live
schema.

Tools are in the `docr` service. Through the hosted DigitalOcean MCP server
invoke them as `docr/<tool>`.

`invoke_tool`'s optional `projection` selects top-level fields of structured
output (for example `["meta"]` or `["manifests"]`). It cannot reach into
array elements, so `manifests.digest` returns nothing; select the whole array
instead. Errors come back as text: `api error: <METHOD> <url>: <status>
(request "<id>") <message>`; quote the request id when escalating.

Risk column is the tier from the skill's safety rules: **Read**, **Confirm**,
**Secret**, **Price**, **Destructive**.

## Registry

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `docr-list` | none | Read | All registries on the account. |
| `docr-get` | `RegistryName` (req) | Read | Name, region, created_at, storage_usage_bytes. Unknown name → 404 "registry does not exist". |
| `docr-options` | none | Read | `available_regions`; `subscription_tiers` with `monthly_price_in_cents`, `included_storage_bytes`, `included_repositories` (`0` = unlimited), and per-account `eligible` / `eligibility_reasons` (`OverRegistryLimit`, `OverRepositoryLimit`, `OverStorageLimit`). |
| `docr-validate-name` | `Name` (req) | Read | Availability check. Names are globally unique. May return 403 when the OAuth grant lacks the scope; fall back to local name rules. |
| `docr-create` | `Name` (req), `SubscriptionTierSlug` (effectively required), `Region` | Price | Omitting `SubscriptionTierSlug` sends an empty slug → 422 "invalid subscription tier slug"; always pass one. Region is permanent. Second registry requires Professional. Can exceed the client timeout (`context deadline exceeded`) yet succeed; check `docr-list` before retrying. |
| `docr-delete` | `RegistryName` (req) | Destructive | Destroys all repositories and images. Irreversible. |
| `docr-docker-credentials` | `RegistryName` (req), `ReadWrite` (bool, default false), `ExpirySeconds` (number, no expiry if unset) | Secret | Returns a Docker `config.json` with an embedded account-wide token. The output enters the session and tool logs, so treat it as exposed; prefer `doctl registries login <registry>` or the control panel. Last resort only, with a short expiry. |

## Repository

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `docr-repository-list` | `RegistryName` (req), `Page` (1), `PerPage` (20), `PageToken` | Read | Only list tool that supports `PageToken`. |
| `docr-repository-tag-list` | `RegistryName` (req), `Repository` (req), `Page` (1), `PerPage` (20) | Read | Tag → manifest digest, size, updated_at. |
| `docr-repository-tag-delete` | `RegistryName` (req), `Repository` (req), `Tag` (req) | Destructive | Removes the tag only. Manifest and blobs remain until GC. |
| `docr-repository-manifest-list` | `RegistryName` (req), `Repository` (req), `Page` (1), `PerPage` (20) | Read | Includes untagged manifests. |
| `docr-repository-manifest-delete` | `RegistryName` (req), `Repository` (req), `Digest` (req, `sha256:...`) | Destructive | Removes the manifest and all its tags. Blobs remain until GC. |

## Subscription (account-wide)

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `docr-subscription-get` | none | Read | Current tier and usage. |
| `docr-subscription-update` | `TierSlug` (req: `starter`, `basic`, `professional`) | Price | Downgrade fails if usage exceeds the target plan. |

## Garbage collection

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `docr-garbage-collection-start` | `RegistryName` (req), `Type` | Destructive | Registry becomes read-only for the duration. `Type` values: `untagged manifests only`, `unreferenced blobs only`, `untagged manifests and unreferenced blobs`. Omitted = `unreferenced blobs only`, which does **not** remove untagged manifests; pass the type explicitly. |
| `docr-garbage-collection-get` | `RegistryName` (req) | Read | The currently active GC. When none is running it returns **404 "did not find an active garbage collection"**; that is the normal idle state, not a failure. |
| `docr-garbage-collection-list` | `RegistryName` (req), `Page` (1), `PerPage` (20) | Read | History with status, blobs_deleted, freed_bytes. |
| `docr-garbage-collection-update` | `RegistryName` (req), `GarbageCollectionUUID` (req), `Cancel` (req, bool) | Confirm | Only supported update is `Cancel: true`. |

## Not covered by MCP tools

Mirror registries, SOCI image push, push-to-deploy for App Platform, and the
control-panel DOKS integration toggle have no `docr-*` tool. Use the docs:
`https://docs.digitalocean.com/products/container-registry/llms.txt`.
