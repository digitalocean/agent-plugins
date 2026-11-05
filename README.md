# DigitalOcean Agent Toolkit

Build, deploy, manage, and troubleshoot applications and infrastructure across
DigitalOcean's cloud platform from compatible AI assistants and agent
platforms.

## Use it

1. Install or add this toolkit using your AI client's plugin instructions. It
   works with any client that supports the open [Agent Plugins](https://agent-plugins.org)
   standard, including Claude Code and Cursor.
2. Connect to the hosted DigitalOcean MCP server and sign in to your
   DigitalOcean account through OAuth.
3. Ask your assistant to manage resources, investigate issues, or answer
   questions about your DigitalOcean environment.

## Install from this marketplace

### Claude Code

Add this repository as a marketplace and install the plugin:

```shell
/plugin marketplace add digitalocean/agent-plugins
/plugin install digitalocean@digitalocean
```

Then run `/mcp` and sign in to your DigitalOcean account when prompted.

### Cursor

To add this repository as a personal marketplace:

1. In the Cursor app, open the Command Palette (`Cmd+Shift+P` on macOS,
   `Ctrl+Shift+P` on Windows and Linux) and run **Open Customize**.
2. Go to **Plugins** and click **Browse Marketplace**.
3. Click **Add Marketplace**, then **Import from GitHub**.
4. Paste `https://github.com/digitalocean/agent-plugins`, leave the scope on
   **User**, and import.

On a Teams or Enterprise plan, an admin can instead import this repository as a team marketplace:

1. Go to **Dashboard > Plugins & MCPs**.
2. In **Team Marketplaces**, click **Add Marketplace**.
3. Use **Import from Repo** and paste
   `https://github.com/digitalocean/agent-plugins`.
4. Add and review plugins using **Add to Marketplace**.
5. Under **Marketplace Settings**, set **Marketplace Access**, optionally
   enable **Auto Refresh**, then save.

Teammates then find and install it from **Customize** in the Cursor sidebar.

## Plugins

Plugins live under [`plugins/`](plugins/), each in its own folder with its
manifests, MCP config, and skills. The repository is indexed for both
toolchains:

| Toolchain | Index |
| --- | --- |
| Claude Code | [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) |
| Cursor | [`.cursor-plugin/marketplace.json`](.cursor-plugin/marketplace.json) |

## Skills

Skills are loaded automatically when a request matches their description.
Each skill carries its own safety rules for the tools it uses.

Browse the available skills in
[`plugins/digitalocean/skills/`](plugins/digitalocean/skills/). Each skill lives
in a folder named in `service-job` format (for example `docr-manage`) and
documents what it does and how to use it in its `SKILL.md`.

## MCP server

| Server       | Endpoint                           |
| ------------ | ---------------------------------- |
| digitalocean | `https://mcp.digitalocean.com/mcp` |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add skills, including the
`service-job` naming convention and the [skill template](templates/SKILL.md).

## License

Licensed under the MIT License, see [LICENSE](LICENSE) for details.
