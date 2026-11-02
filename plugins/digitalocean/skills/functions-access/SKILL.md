---
name: functions-access
description: >-
  Create, list, or delete DigitalOcean Functions namespace access keys for
  third-party tools or CI. Use only when the user explicitly asks for a
  Functions access key, a namespace key, or to revoke one. Do not use for
  ordinary deploy, invoke, or doctl serverless connect.
metadata:
  author: digitalocean
  product: functions
  version: "1.0"
---

# Functions namespace access keys

An access key is a credential for a namespace's data plane. Normal MCP
deploys and invokes do not need one. The server creates and caches its own
keys. `doctl serverless connect <hint>` uses the user's DigitalOcean API
token, not a key you mint here.

Your knowledge of Functions may be outdated. Prefer tool output and the
access-key how-to over pre-training.

## Retrieval sources

| Source | Retrieve via | Use for |
| --- | --- | --- |
| Tool reference | `references/tools.md` in **functions-manage**, or `retrieve_schema` | Arguments |
| Access key how-to | `https://docs.digitalocean.com/products/functions/how-to/manage-namespace-access/index.html.md` | What keys are for |
| Limits | `https://docs.digitalocean.com/products/functions/details/limits/index.html.md` | 200 keys per team, including revoked keys |

## How tools are reached

Tools are in the `functions` service. Through `invoke_tool`, use
`functions/<tool>`. `NamespaceID` is the namespace UUID from
`functions-list-namespaces`.

Creating a key requires the caller's token to include `function:admin`. A
403 means the OAuth grant or API token lacks that scope. Say so. Do not
retry with a different name.

## Safety rules

| Tier | Behaviour | Tools |
| --- | --- | --- |
| Read | Run freely. Output is metadata only (name, id, created, expires). Secrets are not returned | `functions-list-access-keys`, `functions-list-namespaces` |
| Secret | Last resort. The secret appears once, in this session and in tool logs | `functions-create-access-key` |
| Destructive | Confirm the exact key id and name. Never for `mcp-do-*` | `functions-delete-access-key` |

Credential rules:

- **The default answer to "let me deploy" or "connect doctl" is not a new
  key.** Send the user through **functions-deploy**, which calls
  `functions-deployment-guide`. That guide connects `doctl` with the API
  token they already have.
- **Anything `functions-create-access-key` returns is exposed.** The secret
  cannot be fetched again. It is in the model context and the client tool
  log. Writing it to a file through another tool records it there too.
  Mint only after the user explicitly asks for a key and you have said that.
- Always set `ExpiresIn` (`24h`, `7d`, minimum `1h`) unless the user
  explicitly asks for a key that does not expire. Tell them the expiry.
- Never use a name that starts with `mcp-do-`. That prefix is reserved for
  the MCP server. A key you create with it is deleted on the next MCP call
  that touches the namespace.
- Never delete a key whose name starts with `mcp-do-`. The server recreates
  it, and the extra calls count toward the 200-key team limit. Revoked keys
  count toward that limit too.
- Do not delete any other key unless the user confirms that key's name and
  id from a list you just showed.
- `functions-get-namespace` and `functions-list-namespaces` can include a
  `key` field. Treat it as a secret: do not repeat it or store it. It is
  not a substitute for an access key the user asked you to create.
- In unattended runs, do not create or delete keys.

## When the user does want a key

1. `functions-list-namespaces` and confirm the namespace label. Use its UUID.
2. `functions-list-access-keys`. Show names, ids, and expiry. Point out any
   `mcp-do-*` rows as server-owned and not to be deleted.
3. If they are near 200 keys, say so before creating another. The limit
   includes revoked keys. Raising it is a support request, not another
   create call.
4. Confirm name and `ExpiresIn`. Then `functions-create-access-key`.
5. Tell the user the secret is in the session history, that it will not be
   shown again, and that they should copy it into their own secret store.
   Do not write it to disk for them. Do not put it in a repo, a Dockerfile,
   or a chat summary you generate later.

## Revoke

1. List keys. Match the name the user gave to one id.
2. Refuse `mcp-do-*` and explain why.
3. After they confirm that id, `functions-delete-access-key`. The secret
   stops working. The slot can still count toward the 200-key limit until
   support says otherwise. Check the limits page rather than promising a
   freed slot.

## Pitfalls

- Minting a key because `doctl` reported "namespace not found", even
  when the user also asks for a key. That error is an account mismatch.
  Do not create a key. Load **functions-deploy** and follow the deployment
  guide's account check.
- Treating a listed `mcp-do-*` key as leaked and deleting it.
- Omitting `ExpiresIn`, which creates a key that does not expire.
