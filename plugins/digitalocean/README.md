# DigitalOcean

Build, deploy, manage, and troubleshoot applications and infrastructure on
DigitalOcean from your AI assistant. This plugin connects the assistant to the
hosted DigitalOcean MCP server and adds skills that guide it through common
product workflows.

## Skills

Skills load automatically when a request matches their description.

Browse the available skills in [`skills/`](skills/).

## MCP server

This plugin connects to DigitalOcean's hosted MCP server:

| Server       | Endpoint                           |
| ------------ | ---------------------------------- |
| digitalocean | `https://mcp.digitalocean.com/mcp` |

Skills may also read public pages from `docs.digitalocean.com`.

## License

Licensed under the MIT License, see [LICENSE](https://github.com/digitalocean/agent-plugins/blob/main/LICENSE) for details.