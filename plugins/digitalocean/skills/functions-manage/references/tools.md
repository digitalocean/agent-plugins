# Functions MCP tool reference

Source of truth: `pkg/registry/functions/` in
[digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean).
If this table and live `retrieve_schema` output differ, trust the live
schema.

Tools are in the `functions` service. Through the hosted DigitalOcean MCP
server invoke them as `functions/<tool>`.

`NamespaceID` is always the namespace UUID from `functions-list-namespaces`,
not the label.

Risk column is the tier from the skills: **Read**, **Confirm**, **Secret**,
**Destructive**, **Run**. **Run** means the call executes user code and
consumes the Functions allowance.

Namespace list and get responses can include `key`. That value is a
data-plane credential. Do not repeat it or write it down. See
**functions-access**.

## Namespaces

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-namespaces` | none | Read | Label, region, UUID, `api_host`. Omit `key` from summaries. |
| `functions-get-namespace` | `NamespaceID` (req) | Read | Same fields, including `key`. Prefer list unless you need one record. |
| `functions-create-namespace` | `Label` (req), `Region` (req) | Confirm | Region slug such as `nyc1`. At most 10 namespaces per account. |
| `functions-delete-namespace` | `NamespaceID` (req) | Destructive | Deletes the namespace and every function, package, and trigger in it. |

## Access keys

Not part of deploy or invoke. See **functions-access**.

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-access-keys` | `NamespaceID` (req) | Read | Metadata only. Secrets are not returned. `mcp-do-*` names are reserved for the MCP server. |
| `functions-create-access-key` | `NamespaceID` (req), `Name` (req), `ExpiresIn` | Secret | Secret is shown once and lands in the session. Never use the `mcp-do-` prefix. Set `ExpiresIn` (`24h`, `7d`, minimum `1h`) unless the user asked for no expiry. Needs `function:admin`. |
| `functions-delete-access-key` | `NamespaceID` (req), `KeyID` (req) | Destructive | Do not delete `mcp-do-*`. Other names only with an explicit yes. Team limit is 200 keys, including revoked keys. |

## Actions

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-actions` | `NamespaceID` (req), `Limit` (0–200, default 30), `Skip` | Read | `Limit: 0` requests the maximum. |
| `functions-get-action` | `NamespaceID` (req), `ActionName` (req), `PackageName`, `IncludeCode` (default false) | Read | Packaged actions need `PackageName` separately. `IncludeCode` can return secrets embedded in source. |
| `functions-create-or-update-action` | `NamespaceID` (req), `ActionName` (req), `PackageName`, `Kind`, `Code`, `Image`, `Main`, `Components`, `Timeout` (ms, default 60000), `Memory` (MB, default 256), `Logs`, `Annotations`, `Parameters` | Confirm | Always overwrites. Inline path for one file and no dependencies. `Kind` examples: `nodejs:20`, `python:3.11`, `go:default`, `php:default`. `blackbox` uses `Image`. `sequence` uses `Components`. Product limits: timeout ≤ 15 min, memory 128 MB–1 GB. |
| `functions-delete-action` | `NamespaceID` (req), `ActionName` (req), `PackageName` | Destructive | Pass `PackageName` or you delete a different action than the one in the package. |
| `functions-invoke-action` | `NamespaceID` (req), `ActionName` (req), `PackageName`, `Blocking` (default true), `Result` (default false), `InvokeTimeout` (ms), `Payload` | Run | One call, user payload. `InvokeTimeout` default and max are 60000. Longer work: `Blocking: false`, then read the activation. |

## Packages

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-packages` | `NamespaceID` (req), `Limit`, `Skip`, `Public` | Read | `Public` includes shared packages. |
| `functions-get-package` | `NamespaceID` (req), `PackageName` (req) | Read | Actions, parameters, annotations. |
| `functions-create-or-update-package` | `NamespaceID` (req), `PackageName` (req), `Publish`, `Annotations`, `Parameters`, `Binding` | Confirm | `Binding` has `namespace` and `name`. `Publish` makes the package public. |
| `functions-delete-package` | `NamespaceID` (req), `PackageName` (req), `Force` (default false) | Destructive | `Force` is required when the package still has actions. Show those actions first. |

## Triggers

Scheduled triggers only. Preview limits are on the limits page (3 triggers,
not available for App Platform function components).

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-triggers` | `NamespaceID` (req) | Read | |
| `functions-get-trigger` | `NamespaceID` (req), `TriggerName` (req) | Read | Cron is under scheduled details. |
| `functions-create-trigger` | `NamespaceID` (req), `Name` (req), `Function` (req), `Cron` (req), `IsEnabled` (default true), `Body` | Confirm | `Function` for a packaged action is `<package>/<action>`. Five-field cron. |
| `functions-update-trigger` | `NamespaceID` (req), `TriggerName` (req), `IsEnabled`, `Cron`, `Body` | Confirm | Does not rename. |
| `functions-delete-trigger` | `NamespaceID` (req), `TriggerName` (req) | Destructive | A later `doctl serverless deploy` can also drop a trigger that is absent from `project.yml`. |

## Activations

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-list-activations` | `NamespaceID` (req), `FunctionName`, `Limit` (0–200, default 30), `Skip`, `Since`, `Upto`, `IncludeDocs` | Read | `Since` and `Upto` are milliseconds since epoch. `IncludeDocs` adds full records to the list. Records are kept for asynchronous invocations, for 72 hours. |
| `functions-get-activation` | `NamespaceID` (req), `ActivationID` (req) | Read | Full record: response, logs, timing, status. |
| `functions-get-activation-logs` | `NamespaceID` (req), `ActivationID` (req) | Read | Logs only. |
| `functions-get-activation-result` | `NamespaceID` (req), `ActivationID` (req) | Read | Result and status only. |

## Deployment guide

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `functions-deployment-guide` | none | Read | Returns the `doctl serverless deploy` playbook. Call it for multi-file projects, dependencies, `build.sh`, or an existing `project.yml`. Follow it. Do not call it for a one-file function with no dependencies. |

## Not on the hosted Functions MCP

`functions-invoker-deploy-file` and
`functions-invoker-deploy-and-invoke-file` are opt-in
(`SERVICES=functions-invoker`) and talk to a MicroVM invoker. They are not
part of `https://functions.mcp.digitalocean.com/mcp` or the marketplace
router. Do not call them from these skills.

Log forwarding, the control-panel web toggle, and App Platform component
deploys have no `functions-*` tool. Use the docs index:
`https://docs.digitalocean.com/products/functions/llms.txt`.
