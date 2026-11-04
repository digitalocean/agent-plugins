# DOKS MCP tool reference

Source of truth: `pkg/registry/doks/` in
[digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean).
If this table and live `retrieve_schema` output differ, trust the live
schema.

Tools are in the `doks` service. Through the hosted DigitalOcean MCP server
invoke them as `doks/<tool>`. The standalone endpoint is
`https://doks.mcp.digitalocean.com/mcp`, where the tools are exposed directly.

`ClusterID` and `NodePoolID` are the UUIDs returned by `doks-list-clusters`
and `doks-list-nodepools`, not the names. `NodeID` comes from the `nodes`
array on a node pool.

Argument names are PascalCase (`ClusterID`) on every tool except
`doks-create-cluster` and `doks-create-nodepool`. Those two take a raw JSON
body in snake_case (`node_pools`, `auto_scale`). Always call `retrieve_schema`
for them instead of guessing the shape.

`invoke_tool`'s optional `projection` selects top-level fields of structured
output (for example `["cluster"]` or `["node_pools"]`). It cannot reach into
array elements. Errors come back as text, for example
`failed to delete cluster: ...`.

Risk column is the tier from the skill's safety rules: **Read**, **Confirm**,
**Secret**, **Price**, **Destructive**.

## Discovery

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `doks-list-options` | none | Read | Returns `versions` (`slug`, `kubernetes_version`, `supported_features`), `regions` and `sizes` (`slug`, `name`). **No prices.** Take every slug for a create call from here. |

## Clusters

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `doks-list-clusters` | `Page` (1), `PerPage` (20) | Read | Result is under `clusters`. |
| `doks-get-cluster` | `ClusterID` (req) | Read | Includes `status.state` (`provisioning`, `running`, `degraded`, `error`, `upgrading`, `deleted`), `version`, `region`, node pools, `ha`, `auto_upgrade`, `surge_upgrade`, `maintenance_policy`. Use it to verify every cluster write. |
| `doks-create-cluster` | raw JSON body, snake_case. Key fields: `name`, `region`, `version`, `node_pools[]`, `ha`, `auto_upgrade`, `surge_upgrade`, `maintenance_policy`, `vpc_uuid`, `tags` | Price | No field is marked required in the schema. The API needs at least `name`, `region`, `version`, and one node pool. `auto_upgrade` and `surge_upgrade` are always sent, so omitting them means `false`. Omitting `ha` lets the API choose a version-based default, which can be on. Region cannot be changed. Returns before the cluster is ready. |
| `doks-update-cluster` | `ClusterID` (req), `Name`, `Tags`, `MaintenancePolicy` (`StartTime` and `Day`, both required together), `AutoUpgrade`, `SurgeUpgrade` | Confirm | `Day` is a weekday name or `any`. `SurgeUpgrade: false` is dropped from the request, so this tool can turn surge upgrades on but not off. |
| `doks-delete-cluster` | `ClusterID` (req) | Destructive | Plain delete: removes the cluster and its worker nodes. Load balancers and volumes that workloads created are **not** removed and keep billing. Irreversible. |
| `doks-get-cluster-upgrades` | `ClusterID` (req) | Read | Result is under `upgrades`: version `slug` values the cluster can move to. Empty means already current. |
| `doks-upgrade-cluster` | `ClusterID` (req), `VersionSlug` (req) | Confirm | Starts a rolling upgrade and returns at once. The "upgraded to ..." text means accepted, not finished. Control plane API is unavailable for a few minutes and worker nodes are replaced. Not reversible. |

## Node pools

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `doks-list-nodepools` | `ClusterID` (req) | Read | Result is under `node_pools`. No paging arguments. |
| `doks-get-nodepool` | `ClusterID` (req), `NodePoolID` (req) | Read | Count, size, autoscale settings, labels, taints, and the `nodes` array with node IDs and state. |
| `doks-create-nodepool` | `cluster_id` (req), `node_pool_create_request` (req): `name`, `size`, `count`, `auto_scale`, `min_nodes`, `max_nodes`, `tags`, `labels`, `taints` | Price | Snake_case body inside the wrapper. Each taint is an object with `Key`, `Value`, `Effect` (capitalised). Nodes bill once they are ready. |
| `doks-update-nodepool` | `ClusterID` (req), `NodePoolID` (req), `Name`, `Count`, `Tags`, `Labels`, `Taints`, `AutoScale`, `MinNodes`, `MaxNodes` | Confirm. Price when it adds nodes | `MinNodes` and `MaxNodes` are only read when `AutoScale` is also sent. `Taints` must be objects with `Key`, `Value`, `Effect` even though the advertised schema says strings; strings are silently dropped. When `Taints` is omitted the handler sends `"taints":null`, which the API treats as no change. An empty list may also be sent as `null`, so clearing all taints may not work. |
| `doks-delete-nodepool` | `ClusterID` (req), `NodePoolID` (req) | Destructive | Removes every node in the pool and evicts its pods. Irreversible. |
| `doks-delete-node` | `ClusterID` (req), `NodePoolID` (req), `NodeID` (req), `SkipDrain`, `Replace` | Destructive | `Replace: true` creates a new node in its place, so capacity is kept. `SkipDrain: true` deletes without evicting pods first and can drop running workloads. |
| `doks-recycle-nodes` | `ClusterID` (req), `NodePoolID` (req), `NodeIDs` (req, non-empty array) | Confirm | Replaces the listed nodes. The underlying API is marked deprecated in the Go client and in `doctl` (use `replace-node`). Prefer `doks-delete-node` with `Replace: true`. |

## Access

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `doks-get-kubeconfig` | `ClusterID` (req) | Secret | Returns the kubeconfig YAML as text, including an authentication token or certificate. The output enters the session and tool logs. |
| `doks-get-credentials` | `ClusterID` (req) | Secret | Returns `server`, `certificate_authority_data`, `client_certificate_data`, `client_key_data`, `token`, `expires_at`. Same exposure as the kubeconfig. |
