# MicroVMs MCP tool arguments

Source of truth: `pkg/registry/microdroplet/README.md` in [digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean), published at `https://docs.digitalocean.com/reference/mcp/microvms-mcp-tools/index.html.md`. If this file and live `retrieve_schema` output differ, trust the live schema.

`id` on lifecycle, exec, and file tools is the MicroVM UUID. `id` on `microvm-checkpoint-get` and `microvm-checkpoint-delete` is the checkpoint UUID.

## Lifecycle

| Tool | Arguments | Limits |
| --- | --- | --- |
| `microvm-list` | `name`, `region`, `tag_name`, `page` (default 1), `per_page` (default 25, max 200) | Paginated |
| `microvm-get` | `id` (req) |  |
| `microvm-create` | `name` (req), `source` (req), `region` and `size` with `oci_ref`, `networking`, `vpc_uuid`, `http_port`, `ports`, `http_protocol`, `auto_pause`, `auto_resume`, `environment`, `tags` | See below |
| `microvm-pause` | `id` (req) | Sync, idempotent |
| `microvm-resume` | `id` (req) | Sync, idempotent |
| `microvm-delete` | `id` (req) |  |

`microvm-create`:

- `name`: unique in the region. 1–63 lowercase letters, digits, and hyphens.
- `source`: `{ "oci_ref": "…" }` or `{ "checkpoint_id": "…" }`.
- `size`: `{ "cpu": 1, "memory": 2048 }`, memory in MiB. Valid pairs are on the limits page.
- `networking`: `public` (default) or `vpc`. `vpc_uuid` is required with `vpc`.
- `http_port`: default `8080`. `ports`: at most 5, including `http_port`. `http_protocol`: `http` or `http2`.
- `auto_pause`: `{ "idle_timeout" }` only. `enabled: false` is rejected.
- `auto_resume`: default true. Cannot be changed after create.
- `environment`: boot-time key/value. Not returned by the API.

## Exec

| Tool | Arguments | Limits |
| --- | --- | --- |
| `microvm-exec` | `id` (req), `argv` (req, non-empty), `cwd`, `output_encoding` (`utf-8` default, or `base64`) | Team feature `exec_pty`. One-shot, no stdin, about 50 seconds. Response: `stdout`, `stderr`, `exit_code`, optional `truncated` and `output_encoding` |

## Files

| Tool | Arguments | Limits |
| --- | --- | --- |
| `microvm-file-read` | `id` (req), `path` (req, absolute), `encoding` (`utf-8` default, or `base64`) | Team feature `file_transfer`. Max 4 MiB. Non-UTF-8 bytes come back as base64. Response: `path`, `size_bytes`, `sha256`, `encoding`, `content` |
| `microvm-file-write` | `id` (req), `path` (req), `content` (req), `encoding` (`utf-8` default, or `base64` for binary), `mode` (default `0644`), `uid`, `gid` | Team feature `file_transfer`. `uid` and `gid` are set together. A SHA-256 mismatch returns 422 and leaves the file untouched. Response: `path`, `size_bytes`, `sha256`, `mode`, `uid`, `gid` |

## Checkpoints

| Tool | Arguments | Limits |
| --- | --- | --- |
| `microvm-checkpoint-list` | `microvm_id` (optional), `page`, `per_page` |  |
| `microvm-checkpoint-get` | `id` (req, checkpoint UUID) | Status `CHECKPOINT_AVAILABLE` or `CHECKPOINT_FAILED` when done |
| `microvm-checkpoint-create` | `id` (req, MicroVM UUID), `name` | Async |
| `microvm-checkpoint-delete` | `id` (req, checkpoint UUID) | 400 while capture is in progress |
