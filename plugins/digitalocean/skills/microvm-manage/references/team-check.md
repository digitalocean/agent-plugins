# Which team the MCP connection uses

The MicroVMs tools do not return which team the OAuth grant or API token belongs to. `doctl account get` and `doctl compute microvm options` describe the team doctl is logged into, which can differ. A CLI reading can name one team while an MCP create bills another.

Before an MCP create, pause, resume, exec, file call, checkpoint, or delete:

1. When doctl is available, run `microvm-list` and `doctl compute microvm list`.
2. If the same MicroVM id appears in both, they are the same team. `doctl account get` and `options` (sizes, regions, limits, `features`, pricing) then apply to that MCP call.
3. If the ids differ, either command fails, or both lists are empty, stop and ask which team the MCP connection uses.
4. When doctl is not available, ask that before the write.
5. Until the team is confirmed, take size pairs from the limits page, not from `options`.

Never create or paste a token to identify the team.

When doctl makes the call, `doctl account get` is the billed team. Show it and confirm it first. This includes an allow-list create and a file upload or download while MCP is connected. A shared MicroVM id does not cover a create, because the MicroVM does not exist yet.

After a delete, verify with the same client that deleted. `microvm-delete` with `microvm-get` or `microvm-list`. `doctl compute microvm delete` with `doctl compute microvm get` or `list`. `microvm-checkpoint-delete` with `microvm-checkpoint-get` or `microvm-checkpoint-list`. `doctl compute microvm checkpoint delete` with `doctl compute microvm checkpoint get` or `list`. A not-found from the other client is unconfirmed, because it may be another team.
