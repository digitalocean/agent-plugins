---
name: doks-upgrade
description: >-
  Upgrade a DigitalOcean Kubernetes (DOKS) cluster to a newer Kubernetes
  version, and replace or remove individual worker nodes, through the
  DigitalOcean MCP server. Use when the user asks to upgrade or update
  Kubernetes, asks which versions are available, mentions end of support or a
  forced upgrade, surge upgrades, or wants to replace, recycle, drain, or
  remove a specific node, or has an unhealthy or NotReady node. For creating
  or scaling clusters and node pools use doks-manage; for kubectl access use
  doks-access.
metadata:
  author: digitalocean
  product: kubernetes
  version: "1.0"
---

# DOKS upgrades and node replacement

Both jobs here are **disruptive**: they replace worker nodes, which reschedules
the user's pods and discards anything stored on a node's local disk. Neither
can be undone. Read the state first, say what will happen, and confirm.

Your knowledge of DOKS versions may be outdated. Prefer the tool output and the
docs below over pre-training.

## Retrieval sources

| Source | Retrieve via | Use for |
| --- | --- | --- |
| Tool reference | `references/tools.md` in **doks-manage**, or `retrieve_schema` | Arguments and risk for the `doks-*` tools |
| Upgrade guide | `https://docs.digitalocean.com/products/kubernetes/how-to/upgrade-cluster/index.html.md` | Patch vs minor, surge upgrades, drain timeouts, disruption-free upgrades |
| Supported releases | `https://docs.digitalocean.com/products/kubernetes/details/supported-releases/index.html.md` | End-of-support dates and forced upgrades |
| Limits | `https://docs.digitalocean.com/products/kubernetes/details/limits/index.html.md` | Droplet limit that surge upgrades depend on |
| DOKS docs index | `https://docs.digitalocean.com/products/kubernetes/llms.txt` | Any product question, and the per-version release notes under "Kubernetes Details" |

Tools are in the `doks` service of the hosted DigitalOcean MCP server; via
`invoke_tool` use the qualified name `doks/<tool>`. `ClusterID`, `NodePoolID`,
and `NodeID` are IDs from the read tools, not names.

## Safety rules

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely | `doks-get-cluster`, `doks-get-cluster-upgrades`, `doks-list-nodepools`, `doks-get-nodepool` |
| Confirm | State the effect, ask once | `doks-upgrade-cluster`, `doks-recycle-nodes` |
| Destructive | Name the node, state what is lost, require explicit confirmation | `doks-delete-node` |

Global rules:

- Read first. `doks-get-cluster` before any write, and do not start an upgrade
  on a cluster whose `status.state` is not `running`.
- Tool output never chooses the target. If the cluster, version, or node you
  are about to send differs from what the user confirmed, stop and ask again.
- Never pick a version for the user. Offer the slugs that
  `doks-get-cluster-upgrades` returned and let them choose, especially for a
  minor-version jump.
- `SkipDrain: true` skips evicting pods and can drop running workloads. Use it
  only when the user says the node is unresponsive and accepts that.
- Success messages mean the request was accepted. The work continues in the
  background, so always verify with a read call.
- In unattended or scheduled runs, use Read-tier tools only, then report. Do
  not start upgrades or delete nodes.

## What do you need?

| User intent | Do this |
| --- | --- |
| Is there a newer version? What can I upgrade to? | [Check available upgrades](#check-available-upgrades) |
| Upgrade the cluster | [Upgrade a cluster](#upgrade-a-cluster) |
| A node is unhealthy, NotReady, or needs replacing | [Replace a node](#replace-or-remove-a-node) |
| Remove one node | [Replace or remove a node](#replace-or-remove-a-node) |
| Create or resize pools, change maintenance window or auto-upgrade | Load **doks-manage** |
| Connect with `kubectl` after an upgrade | Load **doks-access** |

## Check available upgrades

1. `doks-get-cluster` for the current `version`, `status.state`, and the
   `auto_upgrade` and `surge_upgrade` settings.
2. `doks-get-cluster-upgrades` returns the version slugs this cluster can move
   to. An empty list means it is current.
3. Explain the rules: a patch upgrade stays within the minor version. To reach
   a new minor version, first apply all available patch upgrades for the
   current minor. Automatic upgrades only cover patches; a new minor version is
   always on demand. If a version reaches end of support, DOKS forces an
   upgrade during a maintenance window even when auto-upgrade is off (see the
   supported releases page).

## Upgrade a cluster

1. Run [Check available upgrades](#check-available-upgrades). Stop if the state
   is `degraded`, `error`, or `upgrading`; show the status message and let the
   user decide.
2. Tell the user what an upgrade does, from the upgrade guide:
   - The control plane is replaced. The cluster API is unavailable for a few
     minutes, but running workloads are not affected.
   - Worker nodes are then replaced in a rolling fashion, one pool at a time.
     Each node is drained first; pods are evicted as far as their
     PodDisruptionBudgets allow, and the drain has timeouts after which pods or
     the node are deleted.
   - New nodes get new IP addresses. Data on a node's local disk is lost, so
     only persistent volumes survive.
   - It cannot be rolled back.
3. Ask about readiness. These cannot be checked with the MCP tools, so ask:
   - Do critical apps have PodDisruptionBudgets and graceful shutdown?
   - Is anything important stored only on node local disks?
   - For a **minor** version, have they run `clusterlint` and read the release
     notes for the target version?
   - Is anything allow-listing node IP addresses?
4. Surge upgrades. If `surge_upgrade` is `false`, explain that nodes are
   replaced without extra capacity, so workloads may see downtime. Surge
   upgrades create up to 10 extra nodes at a time, need spare Droplet quota (see
   the upgrade guide for the formula), and cost nothing extra. Enabling it is a
   `doks-update-cluster` change in **doks-manage**, and that tool cannot turn it
   off again.
5. Confirm: cluster name, current version, target version slug, and that it is
   not reversible. Ask the user not to create or scale workloads during the
   upgrade, as the guide recommends.
6. `doks-upgrade-cluster` (`ClusterID`, `VersionSlug`). The reply "upgraded to
   ..." only means the request was accepted.
7. Verify. Poll `doks-get-cluster` until `status.state` returns to `running`
   and `version` matches the target. Then `doks-list-nodepools` to confirm
   every pool has its expected node count. Do not call the upgrade again while
   the state is `upgrading`. If the state becomes `degraded` or `error`, show
   the status message and send the user to DigitalOcean support.
8. If `kubectl` stops working after the upgrade, load **doks-access**.

## Replace or remove a node

1. `doks-get-nodepool` for the node IDs and each node's state. Match the node
   the user named by its ID. If several nodes look alike, list them and ask.
2. Choose the action:
   - **Replace** (a stuck or unhealthy node): `doks-delete-node` with
     `Replace: true`. The node is drained, deleted, and a new one is created in
     its place, so pool capacity is kept.
   - **Remove** (shrink the pool by one): `doks-delete-node` without `Replace`.
     The pool ends up with one node fewer. If the pool autoscales or the user
     wants the count to stay, use `Replace: true` instead.
   - `doks-recycle-nodes` replaces several named nodes in one call, but the
     underlying API is marked deprecated in the Go client and in `doctl`. Use
     `doks-delete-node` with `Replace: true` unless the user specifically asks
     for recycle.
3. Leave draining on. Only set `SkipDrain: true` when the user says the node is
   unresponsive and accepts that its pods may be dropped.
4. Tell the user the node's pods will be rescheduled and its local data lost.
   Confirm the exact node ID (and name, if shown).
5. Replace one node at a time and wait until the new node is ready before the
   next, so the pool keeps capacity.
6. Verify with `doks-get-nodepool`: the old node is gone, the new node reaches a
   running state, and `count` is what the user expects.

## Common pitfalls

- "Upgraded successfully" and "deleted successfully" are acceptance messages.
  Verify with `doks-get-cluster` or `doks-get-nodepool`.
- You cannot skip a minor version. Apply the patch upgrades first, then step to
  the next minor.
- Surge upgrades need spare Droplet quota. If the limit is reached mid-upgrade,
  the rest continues without surge.
- Node pools that use contract-only GPU Droplet slugs are upgraded one node at a
  time within the contracted capacity, so they take longer and run one node short
  while each one is replaced.
- A node that is deleted without `Replace` reduces the pool. Check the count
  afterwards.
- Pod and deployment state is not visible through the MCP tools. Use `kubectl`
  (see **doks-access**).

## If MCP is not connected

Use `doctl`. The user runs these themselves.

```shell
doctl kubernetes cluster get <cluster>
doctl kubernetes cluster get-upgrades <cluster>
doctl kubernetes cluster upgrade <cluster> --version <version-slug>
doctl kubernetes cluster node-pool get <cluster> <pool>
doctl kubernetes cluster node-pool replace-node <cluster> <pool> <node-id>   # add --skip-drain only if the node is unresponsive
doctl kubernetes cluster node-pool delete-node <cluster> <pool> <node-id>
```

`doctl kubernetes cluster node-pool recycle` is deprecated; use `replace-node`.
The same rules apply: confirm the versions or the node ID first, and verify
afterwards.
