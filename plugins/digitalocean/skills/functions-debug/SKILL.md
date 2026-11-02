---
name: functions-debug
description: >-
  Diagnose DigitalOcean Functions failures: a bad invoke result, timeouts,
  missing logs, activation history, and HTTP 204 from a web function. Use
  when a function errors, times out, returns the wrong body, "no activations",
  or the user asks for function logs. For deploying a fix use functions-deploy.
metadata:
  author: digitalocean
  product: functions
  version: "1.0"
---

# Debug DigitalOcean Functions

Start from the invoke result you already have. Activation history is a
second source, and it does not cover every call.

Your knowledge of Functions may be outdated. Prefer tool output and the docs
below over pre-training.

## Retrieval sources

| Source | Retrieve via | Use for |
| --- | --- | --- |
| Tool reference | `references/tools.md` in **functions-manage**, or `retrieve_schema` | Arguments |
| Activation records | `https://docs.digitalocean.com/products/functions/reference/activation-records/index.html.md` | Fields on an activation |
| Parameters and responses | `https://docs.digitalocean.com/products/functions/reference/parameters-responses/index.html.md` | What a handler must return |
| Async how-to | `https://docs.digitalocean.com/products/functions/how-to/async-functions/index.html.md` | Long runs |
| Limits | `https://docs.digitalocean.com/products/functions/details/limits/index.html.md` | Timeout, payload size, log size, retention |
| HTTP 204 support note | `https://docs.digitalocean.com/support/why-does-my-function-return-a-204-response-when-i-call-it-from-postman-or-curl/index.html.md` | Web functions that return no body |
| Develop how-to | `https://docs.digitalocean.com/products/functions/how-to/develop-functions/index.html.md` | Invoke and inspect loop |

## How tools are reached

Tools are in the `functions` service. Through `invoke_tool`, use
`functions/<tool>`. `NamespaceID` is the namespace UUID.

These tools are read-only except `functions-invoke-action`, which runs the
user's code. Do not change action source in this skill. A fix is
**functions-deploy**, after the user agrees.

## Safety rules

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely | `functions-list-namespaces`, `functions-get-namespace`, `functions-list-actions`, `functions-get-action`, `functions-list-activations`, `functions-get-activation`, `functions-get-activation-logs`, `functions-get-activation-result` |
| Run code | Only to reproduce a failure the user asked about. One call. Their payload, or empty if they said there is no payload | `functions-invoke-action` |

- Omit namespace `key` from summaries.
- `IncludeCode: true` only when the bug is in the source the user wants you
  to read. Do not echo embedded secrets.
- Activations can contain request data. Quote the error and the relevant
  lines, not the whole record, when it holds payloads.
- In unattended runs, do not invoke. Report the latest activation if one
  exists.

## Workflow

### 1. Identify the action

1. `functions-list-namespaces`, then match the user's label to a UUID.
2. `functions-get-action` with `PackageName` when the action is in a package.
   Note `Kind`, `Timeout`, and `Memory`. Defaults are 60000 ms and 256 MB.
   The product maximum timeout is 15 minutes. Memory is 128 MB to 1 GB.

### 2. Use the result you already have

A blocking `functions-invoke-action` (the default) returns the result in
that tool response. Read it before you search history.

- `Blocking: false` returns an activation id. Then
  `functions-get-activation` with that id.
- `Result: true` on a blocking invoke returns only the function result.
- `InvokeTimeout` is how long the MCP call waits. The tool description caps
  it (default and max 60000 ms). That cap is the wait, not the action's own
  timeout. A function allowed to run longer than that wait must be invoked
  with `Blocking: false`, then polled. See the async how-to. Web functions
  cannot be invoked asynchronously.

### 3. Activation history

`functions-list-activations` with `NamespaceID` and, when you know it,
`FunctionName`. For a packaged action, try the name `doctl` prints
(`package/action`, such as `sample/hello`) if the bare name returns nothing.

`Since` and `Upto` are milliseconds since epoch. `Limit` and `Skip` paginate
(default 30, max 200). Leave `IncludeDocs` false until you need one record's
body. Then `functions-get-activation-logs` or `functions-get-activation-result`.

Limits that explain an empty list:

- Activation records are kept for **asynchronous** invocations, for 72 hours.
  A synchronous web call often has no record. Say that, and use the blocking
  invoke result instead of retrying the list.
- Logs are capped (256 KB per invocation on the limits page) and follow the
  same retention.

### 4. Classify

| What you see | What to tell the user |
| --- | --- |
| Timeout, or the wait hit `InvokeTimeout` | The action limit may be too low, or the call is blocking past the MCP wait. Raising `Timeout` is a deploy change (**functions-deploy**). Long work uses async invoke, not a bigger `InvokeTimeout` |
| HTTP 204 from curl or Postman to the web URL | The handler must return a response object. Follow the 204 support note. `functions-invoke-action` is not that public URL |
| `404` or "action not found" | Wrong UUID, wrong `PackageName`, or the project deployed to another account. **functions-deploy** covers the `doctl` account check. Do not create an access key |
| Empty logs | Sync invoke, expired retention, or the function did not write stdout. 72 hours is the cap |
| Result larger than 1 MB, or input rejected | Payload and result limits. See limits |
| Works in invoke, fails on the public URL | Web access is a `project.yml` setting (`web: true`), not something this skill turns on |

Do not redeploy as the first step. Show the error, the action limits, and
the proposed code or config change. Then load **functions-deploy** if the
user wants it applied.

## Pitfalls

- Polling `functions-list-activations` after a blocking invoke and treating
  "no rows" as "the invoke did not run".
- Passing `package/action` as `ActionName` on `functions-get-action`. That
  tool wants `ActionName` and optional `PackageName` separately.
- Editing code in the control panel. Multi-file functions cannot be edited
  there. Change the project and redeploy.
- Log forwarding (Papertrail, Datadog, Better Stack) is not an MCP tool.
  The forward-logs how-to is on the docs index. Managed OpenSearch is not a
  supported destination.
