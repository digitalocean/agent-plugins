# MicroVMs MCP tool reference

Source of truth: `pkg/registry/microdroplet/README.md` in
[digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean).
The published page is
`https://docs.digitalocean.com/reference/mcp/microvms-mcp-tools/index.html.md`.
If this table and live `retrieve_schema` output differ, trust the live
schema.

Tools are in the `microvms` service. Through the hosted DigitalOcean MCP
server invoke them as `microvms/<tool>`. On
`https://microvms.mcp.digitalocean.com/mcp` they are exposed as `microvm-*`.
Hosted agents may see `digitalocean_microvm-*`.

`id` on lifecycle, exec, and file tools is the MicroVM UUID. `id` on
`microvm-checkpoint-get` and `microvm-checkpoint-delete` is the checkpoint
UUID.

Risk column is the tier from **microvm-manage**: **Read**, **Confirm**,
**Price**, **Run**, **Destructive**.

## Lifecycle

| Tool             | Arguments                                                                                                                                                                             | Risk        | Notes                                                                                                                                                                                                                                                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `microvm-list`   | `name`, `region`, `tag_name`, `page` (default 1), `per_page` (default 25, max 200)                                                                                                    | Read        | Paginated.                                                                                                                                                                                                                                                                                                                                                                              |
| `microvm-get`    | `id` (req)                                                                                                                                                                            | Read        | After `microvm-create`, poll until `state` is `running` or `failed`, and on `failed` report `failure_reason`. Any other get is one read. `paused` and `running` are finished states.                                                                                                                                                                                                    |
| `microvm-create` | `name` (req), `source` (req), `region` and `size` with `oci_ref`, `networking`, `vpc_uuid`, `http_port`, `ports`, `http_protocol`, `auto_pause`, `auto_resume`, `environment`, `tags` | Price       | `source` is exactly one of `oci_ref` or `checkpoint_id`. Restore inherits region and size. `name` is unique in the region. `networking` is `public` (default) or `vpc`. `http_port` defaults to `8080`. `ports` max 5 and includes `http_port`. `auto_pause` is `{ "idle_timeout" }` only; `enabled: false` is rejected. `auto_resume` defaults to true. No egress-allow-list argument. |
| `microvm-pause`  | `id` (req)                                                                                                                                                                            | Confirm     | Sync and idempotent. Compute billing stops. Disk storage continues.                                                                                                                                                                                                                                                                                                                     |
| `microvm-resume` | `id` (req)                                                                                                                                                                            | Confirm     | Sync and idempotent. Compute billing starts again.                                                                                                                                                                                                                                                                                                                                      |
| `microvm-delete` | `id` (req)                                                                                                                                                                            | Destructive | Irreversible. Confirm the exact id first.                                                                                                                                                                                                                                                                                                                                               |

`size` is `{ "cpu": 1, "memory": 2048 }` with memory in MiB. Valid pairs are
on the limits page, not in this file. `environment` is boot-time key/value
and is not returned by the API. Do not put secrets in it unless the user
supplied them for that create.

There is no resize tool. Console WebSocket and the Images API are not MCP
tools.

## Exec

Needs the team feature `exec_pty`. One-shot, non-interactive, no stdin. About
50 seconds. HTTP 200 with a non-zero `exit_code` is still a successful tool
call.

| Tool           | Arguments                                                                                    | Risk | Notes                                                                                                                                                                                                                                                               |
| -------------- | -------------------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `microvm-exec` | `id` (req), `argv` (req, non-empty), `cwd`, `output_encoding` (`utf-8` default, or `base64`) | Run  | Show the id and the full argv. Only a command the user asked for. When `auto_resume` is true, exec resumes a paused MicroVM and compute billing starts. Response: `stdout`, `stderr`, `exit_code`, optional `truncated` and `output_encoding`. Do not echo secrets. |

## Files

Needs the team feature `file_transfer`. A paused MicroVM is resumed, which
restarts compute billing. Say so and get a yes first.

| Tool                 | Arguments                                                                                    | Risk | Notes                                                                                                                                                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `microvm-file-read`  | `id` (req), `path` (req), `encoding` (`utf-8` default, or `base64`)                          | Run  | Absolute path. Max 4 MiB. Non-UTF-8 bytes come back as base64. Check `encoding` before treating `content` as text. Response: `path`, `size_bytes`, `sha256`, `encoding`, `content`.                                                      |
| `microvm-file-write` | `id` (req), `path` (req), `content` (req), `encoding`, `mode` (default `0644`), `uid`, `gid` | Run  | Replaces the target. Parent directory must exist. `uid` and `gid` are set together. SHA-256 is sent with the body; a mismatch returns 422 and leaves the file untouched. Response: `path`, `size_bytes`, `sha256`, `mode`, `uid`, `gid`. |

A read above 4 MiB is `doctl compute microvm download` (stream, up to 64 MiB).
A write that does not fit `microvm-file-write` is `doctl compute microvm upload`
(up to 64 MiB). Download cannot place a file on the guest. Read the subcommand
`--help` before calling either one. Do not pass `--access-token` or `--show`.

## Checkpoints

| Tool                        | Arguments                        | Risk        | Notes                                                                                                                                                                                                                                                                       |
| --------------------------- | -------------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `microvm-checkpoint-list`   | `microvm_id`, `page`, `per_page` | Read        | `microvm_id` is optional.                                                                                                                                                                                                                                                   |
| `microvm-checkpoint-get`    | `id` (req, checkpoint UUID)      | Read        | Poll until `CHECKPOINT_AVAILABLE` or `CHECKPOINT_FAILED`.                                                                                                                                                                                                                   |
| `microvm-checkpoint-create` | `id` (req, MicroVM UUID), `name` | Confirm     | Async. Stores memory and disk, which can include guest secrets. Storage stays billable until this checkpoint is deleted, including after the MicroVM is deleted.                                                                                                            |
| `microvm-checkpoint-delete` | `id` (req, checkpoint UUID)      | Destructive | 400 while capture is still in progress. That restriction does not apply to `microvm-delete`. Verify with `microvm-checkpoint-get` or `microvm-checkpoint-list`. A `microvm-get` not-found does not show the checkpoint is gone, and its storage stays billable until it is. |

## Not on these tools

The published doctl reference does not list an egress allow list. If
`doctl compute microvm create --help` prints those flags, use only the flags
it prints and confirm every destination and port first. A read above 4 MiB
is `doctl compute microvm download`. A write that does not fit
`microvm-file-write` is `doctl compute microvm upload` (up to 64 MiB). Read
that subcommand's `--help` before calling it. Download cannot place a file
on the guest.

Never paste `DIGITALOCEAN_ACCESS_TOKEN` into a command or the chat.
