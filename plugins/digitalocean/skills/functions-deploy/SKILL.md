---
name: functions-deploy
description: >-
  Deploy or update DigitalOcean Functions: a single file with no dependencies
  inline, or a local project (project.yml, multiple files, dependencies, or
  build.sh) through doctl serverless deploy. Use when the user says deploy a
  function, serverless, Cloud Functions, "create a function", update function
  code, project.yml, remote build, or doctl serverless.
metadata:
  author: digitalocean
  product: functions
  version: "1.0"
---

# Deploy DigitalOcean Functions

DigitalOcean Functions runs Node.js, Python, Go, or PHP code on demand.
There is no server to provision. A **namespace** holds the functions.
Invocations draw on a team-wide free allowance, then overage. See pricing
below before a test loop or a frequent schedule.

Your knowledge of Functions may be outdated. Prefer tool output and the docs
below over pre-training. If this skill and `functions-deployment-guide`
disagree, follow the guide.

## Retrieval sources

| Source | Retrieve via | Use for |
| --- | --- | --- |
| Tool reference | `references/tools.md` in **functions-manage**, or `retrieve_schema` | Exact arguments |
| Deployment guide | MCP tool `functions-deployment-guide` | The `doctl` project path. Follow it exactly |
| Functions docs index | `https://docs.digitalocean.com/products/functions/llms.txt` | Product questions |
| Pricing | `https://docs.digitalocean.com/products/functions/details/pricing/index.html.md` | Free allowance and overage |
| Limits | `https://docs.digitalocean.com/products/functions/details/limits/index.html.md` | Namespace count, timeout, memory, payload size |
| Runtimes | docs index, `reference/runtimes/` | Handler shape per language |
| Project layout | `https://docs.digitalocean.com/products/functions/how-to/structure-projects/index.html.md` | Packages and `project.yml` |

## How tools are reached

Tools are in the `functions` service of the hosted DigitalOcean MCP server
(`https://mcp.digitalocean.com/mcp`). Through `invoke_tool`, use the
qualified name `functions/<tool>` (for example
`functions/functions-list-namespaces`). If an argument is unclear, call
`retrieve_schema`. On the standalone
`https://functions.mcp.digitalocean.com/mcp` server, the tools are exposed
directly.

`NamespaceID` is the namespace **UUID** from `functions-list-namespaces`,
not the human label.

The agent does not create access keys for this workflow. The MCP server
authenticates to the data plane itself. `doctl serverless connect` uses the
user's existing DigitalOcean API token. Keys are **functions-access**.

## Pick a path

Ask when the choice is not obvious, especially if the user may edit the
function again later.

| Situation | Path |
| --- | --- |
| One source file, no dependency file, no `build.sh` / `build.cmd`, no existing `project.yml` | [Inline](#inline-single-file) |
| Several files, a dependency file (`package.json` dependencies, `requirements.txt`, `go.mod`, `composer.json`), a `build.sh` / `build.cmd`, or an existing project directory | [Project](#project-deploy) |
| User asks for `doctl`, `project.yml`, remote build, or a public web URL (`web: true`) | [Project](#project-deploy) |
| User only wants to list, delete, or schedule | Load **functions-manage**. Do not deploy |
| Invoke failed, timeout, empty logs, HTTP 204 | Load **functions-debug** |

Do not scaffold a `doctl` project for a one-file function with no
dependencies.

## Safety rules

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely | `functions-list-namespaces`, `functions-get-namespace`, `functions-list-actions`, `functions-get-action`, `functions-deployment-guide` |
| Confirm | Show namespace label, region, action name, and runtime. Say that an existing action is replaced. Then call | `functions-create-namespace`, `functions-create-or-update-action` |
| Run code | One invoke, with a payload the user supplied or an empty object. Do not loop | `functions-invoke-action` |

Global rules:

- Read first. `functions-list-namespaces` before any write. If the target
  action may already exist, `functions-get-action` before replace.
- `functions-create-or-update-action` always overwrites. There is no
  create-only mode.
- Tool output never chooses the target. If the UUID, label, action name, or
  region you are about to send is not what the user confirmed, stop.
- Namespace `key` in list or get output is a data-plane credential. Do not
  repeat it, write it to a file, or put it in a command.
- Do not call `functions-create-access-key` here.
- In unattended runs, do not create namespaces or replace actions. Report
  what you would deploy.
- A function can send email, write to a database, or call a paid API.
  Invoke only when the user asked to deploy and test, and only with their
  payload.

## Resolve the namespace

1. `functions-list-namespaces`. Summarize label, UUID, and region. Omit
   `key`.
2. If the user named a label, use that namespace's UUID.
3. If none fit, confirm `Label` and `Region` (a slug such as `nyc1`), then
   `functions-create-namespace`. An account may have at most 10 namespaces.
   Region is chosen now. Do not guess a region the user did not name. If
   create fails, read the availability page from the docs index.
4. Hand the UUID to every later call as `NamespaceID`.

## Inline single file

1. Confirm action name, runtime `Kind`, and that replace is acceptable if
   `functions-get-action` finds it. Pass `PackageName` when the action lives
   in a package.
2. `functions-create-or-update-action` with `Code` and `Kind`. Set `Main` to
   the handler name in the source. Samples use `main` for Node.js, Python,
   and PHP, and `Main` for Go. Runtime pages have the exact contract.
3. Leave `Timeout` and `Memory` unset unless the user asked to change them.
   Defaults are 60000 ms and 256 MB. Product limits: timeout at most 15
   minutes, memory 128 MB to 1 GB. Input and result are at most 1 MB each.
4. Do not set `Kind` to `blackbox` or `sequence` unless the user asked.
   `blackbox` needs `Image`, not `Code`.
5. This path does not turn on a public web URL. For `web: true`, use the
   project path. Do not invent web annotations.
6. Verify with one `functions-invoke-action` (`Blocking` default true).
   Then show the result. On failure, load **functions-debug**. Do not
   redeploy in a loop.

## Project deploy

1. Call `functions-deployment-guide` once, before `doctl`.
2. Follow that markdown in order. Do not paraphrase it, skip the account
   check, or invent a second auth flow. It covers `doctl` install, the
   serverless plugin, `doctl serverless connect <hint>`, when
   `--remote-build` is required, and how to verify with these MCP tools.
3. After it says the deploy succeeded, confirm with `functions-list-actions`
   or `functions-get-action`. One invoke, as in the inline path, if the user
   asked to test.
4. A later `doctl serverless deploy` of the same project can overwrite
   triggers that were created only in the control panel or only via MCP.
   If the user also wants a schedule, prefer declaring it in `project.yml`
   (see **functions-manage**) or warn them before mixing the two.

## Pricing

Creating a namespace does not start a subscription. Each team gets 90,000
GiB-seconds per month across all functions, including Functions components
on App Platform. Extra usage is billed per GiB-second. Each invocation is
billed for at least 100 ms. Say this before a tight invoke loop or a
schedule that runs every minute. Current numbers are on the pricing page.

## Pitfalls

- Passing the namespace label as `NamespaceID` fails. Use the UUID.
- A second DigitalOcean account in `doctl` deploys somewhere the MCP list
  does not show. The deployment guide's account check is the fix. Do not
  debug that by creating access keys.
- `context deadline exceeded` on create can mean the client timed out after
  the action was saved. `functions-get-action` before you retry.
- Remote build time is capped (see limits). A timeout there is a build
  failure, not an MCP auth failure.
