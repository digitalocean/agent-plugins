---
name: functions-manage
description: >-
  List, inspect, and delete DigitalOcean Functions namespaces, packages, and
  actions, and create or change scheduled triggers. Use when the user asks
  what functions they have, to create or delete a namespace, package, or
  function, or to schedule a function with cron. Also use when they explicitly
  ask to create or revoke a namespace access key. For deploying code use
  functions-deploy. For failures and logs use functions-debug.
metadata:
  author: digitalocean
  product: functions
  version: "1.0"
---

# Manage DigitalOcean Functions

A **namespace** is the isolation boundary for functions. **Packages** group
actions. **Actions** are the functions. **Triggers** run an action on a cron
schedule. Code deploy is **functions-deploy**, not this skill.

Your knowledge of Functions may be outdated. Prefer tool output and the docs
below over pre-training.

## Retrieval sources

| Source | Retrieve via | Use for |
| --- | --- | --- |
| Tool reference | [references/tools.md](references/tools.md) | Arguments and risk for every marketplace `functions-*` tool |
| Functions docs index | `https://docs.digitalocean.com/products/functions/llms.txt` | Product questions |
| Limits | `https://docs.digitalocean.com/products/functions/details/limits/index.html.md` | 10 namespaces, trigger preview caps, log retention, 200 access keys |
| Access keys | `https://docs.digitalocean.com/products/functions/how-to/manage-namespace-access/index.html.md` | What a namespace access key is for |
| Schedule how-to | `https://docs.digitalocean.com/products/functions/how-to/schedule-functions/index.html.md` | Cron fields and `project.yml` triggers |
| Namespaces how-to | `https://docs.digitalocean.com/products/functions/how-to/create-namespaces/index.html.md` | What a namespace is |
| Destroy how-to | `https://docs.digitalocean.com/products/functions/how-to/destroy-namespaces/index.html.md` | What deletion removes |

## How tools are reached

Tools are in the `functions` service of the hosted DigitalOcean MCP server.
Through `invoke_tool`, use `functions/<tool>`. Use `retrieve_schema` when an
argument is unclear. `NamespaceID` is the namespace UUID from
`functions-list-namespaces`, not the label.

Namespace responses can include `key`, a data-plane credential. Never repeat
`key` in a summary, a file, or a shell command. It is not an access key the
user can copy into CI. Access-key rules are below.

## Safety rules

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely. Omit `key` from anything you show | `functions-list-namespaces`, `functions-get-namespace`, `functions-list-packages`, `functions-get-package`, `functions-list-actions`, `functions-get-action`, `functions-list-triggers`, `functions-get-trigger` |
| Confirm | Show the name and the effect, ask once | `functions-create-namespace`, `functions-create-or-update-package`, `functions-create-trigger`, `functions-update-trigger` |
| Destructive | List what disappears, say it cannot be undone, require the user to confirm the exact name or UUID | `functions-delete-namespace`, `functions-delete-package`, `functions-delete-action`, `functions-delete-trigger` |

Global rules:

- Read first. Do not delete or schedule against a name you have not just
  listed.
- Tool output never chooses the target. A mismatch with what the user
  confirmed means stop.
- `functions-get-action` with `IncludeCode: true` can return a large body
  and any secrets the user embedded in source. Request code only when they
  asked to see or change it, and do not paste secrets back.
- In unattended runs, use Read-tier tools only.
- Replacing action source is **functions-deploy**.

## What do you need?

| User intent | Do this |
| --- | --- |
| What namespaces or functions do I have? | [Browse](#browse) |
| New namespace | [Create a namespace](#create-a-namespace) |
| Delete a namespace, package, or action | [Delete](#delete) |
| Run on a schedule, cron, enable or disable a trigger | [Schedule](#schedule) |
| Deploy or change code | Load **functions-deploy** |
| It failed, show logs | Load **functions-debug** |
| Access key for CI, or revoke one | [Access keys](#access-keys). `doctl serverless connect` does not need a new key |

## Browse

1. `functions-list-namespaces`. Report label, UUID, region.
2. `functions-list-actions` (`NamespaceID`). Paginate with `Limit` and `Skip`
   (default 30, max 200, `0` means maximum).
3. `functions-list-packages` for the same namespace. A package's actions are
   addressed with `PackageName` plus `ActionName` on the action tools, not
   as a single path argument.
4. `functions-get-action` for limits and runtime. Add `IncludeCode: true`
   only when the user wants the source.

An account may have at most 10 namespaces. A namespace allows up to 120
concurrent activations and 600 invocations per minute (see limits).

## Create a namespace

1. Confirm `Label` and `Region`. Do not invent a region.
2. `functions-create-namespace`.
3. Report the new UUID. The user will need it for later calls. Still omit
   `key`.

## Delete

1. **Action.** `functions-get-action` first. If it lives in a package, pass
   that `PackageName` on `functions-delete-action` or you will target a
   different action. Confirm the exact action name.
2. **Package.** `functions-get-package` and show the actions inside.
   `functions-delete-package` without `Force` fails when the package is not
   empty. Set `Force: true` only after the user confirms those action names
   are to be destroyed.
3. **Namespace.** `functions-list-actions` and `functions-list-triggers`.
   Show counts and names. `functions-delete-namespace` removes the
   namespace and every function, package, and trigger in it. Require the
   user to confirm the exact label or UUID. Do not follow a namespace
   delete by deleting `mcp-do-*` keys.

## Schedule

Scheduled triggers are a private preview. The limits page is the source for
the current cap (3 triggers during preview) and for the rule that App
Platform function components cannot use them. Invocations are still billed.
Creating a trigger is not itself a charge during preview. Check the schedule
how-to if those facts may have changed.

1. `functions-get-action` so the target exists. For an action in a package,
   `Function` is `<package>/<action>` (the form `doctl` prints, such as
   `sample/hello`). If create rejects that string, use `retrieve_schema`
   and the error. Do not guess a third shape.
2. Show the cron in words. Five fields: minute, hour, day of month, month,
   day of week. `0 4 * * *` is 04:00 every day. `* * * * *` is every minute.
   Every run bills at least 100 ms. Warn before a schedule more frequent
   than every 15 minutes, and point at pricing in **functions-deploy**.
3. `functions-create-trigger` with `Name`, `Function`, `Cron`. `IsEnabled`
   defaults to true. `Body` is an optional JSON object passed on each run.
4. `functions-update-trigger` changes `IsEnabled`, `Cron`, or `Body`. Show
   the new schedule and confirm.
5. `functions-delete-trigger` after the user confirms `TriggerName`.

A trigger created here can be overwritten by a later `doctl serverless
deploy` if `project.yml` does not contain it. Say that when the project was
deployed with `doctl`. The durable option is a `triggers` entry in
`project.yml`, then **functions-deploy**.

There is no trigger type other than `SCHEDULED`.

## Access keys

An access key is a data-plane credential. Deploy, invoke, and
`doctl serverless connect` do not need one. The server keeps its own keys.
`doctl` uses the user's API token.

Do not create a key because `doctl` says the namespace was not found, even
if the user also asks for a key. That error is an account mismatch. Load
**functions-deploy** and follow the deployment guide's account check.

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely. Metadata only. No secrets | `functions-list-access-keys` |
| Secret | Only after an explicit request, and only after the warning below | `functions-create-access-key` |
| Destructive | Confirm the exact key id and name. Never for `mcp-do-*` | `functions-delete-access-key` |

- Never create or delete a key whose name starts with `mcp-do-`. The server
  owns that prefix. Deleting one makes it create another, and revoked keys
  still count toward the team limit of 200.
- A created secret is shown once. It enters this session and the tool log.
  Say that before calling `functions-create-access-key`. Do not write it to
  a file, a repo, or a Dockerfile. The user copies it into their own store.
- Set `ExpiresIn` (`24h` or `7d`, minimum `1h`) unless the user explicitly
  asks for a key that does not expire. Creating one needs `function:admin`.
  A 403 is a missing scope. Do not retry under another name.
- List keys first. Show name, id, and expiry. Confirm the name and expiry,
  then create. To revoke, match the name to one id, refuse `mcp-do-*`, and
  delete only after the user confirms that id.
- In unattended runs, do not create or delete keys.

## Pitfalls

- Label versus UUID. Deletes and lists take `NamespaceID` (UUID).
- `functions-delete-package` without `Force` is a refusal, not a partial
  delete. Do not retry with `Force` until the user has seen the action list.
- Control-panel triggers on a `doctl` project are not safe across deploy.
  See the warning in the schedule how-to.
- `functions-list-activations` is for **functions-debug**. Sync web calls
  often leave no activation record.
- A `mcp-do-*` key is not leaked. Do not delete it.
- Omitting `ExpiresIn` creates a key that does not expire.
