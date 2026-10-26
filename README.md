# DigitalOcean Agent Toolkit

Build, deploy, manage, and troubleshoot applications and infrastructure across
DigitalOcean's cloud platform from compatible AI assistants and agent
platforms.

## Use it

1. Install or add this toolkit using your AI client's plugin instructions.
2. Connect to the hosted DigitalOcean MCP server and sign in to your
   DigitalOcean account through OAuth.
3. Ask your assistant to manage resources, investigate issues, or answer
   questions about your DigitalOcean environment.

The toolkit reads resources freely, asks for confirmation before making
changes, and shows the price before creating billable resources.

## MCP server

| Server       | Endpoint                           |
| ------------ | ---------------------------------- |
| digitalocean | `https://mcp.digitalocean.com/mcp` |

## Data

Your AI client sends requests and the required resource details to
`mcp.digitalocean.com`, authenticated with your OAuth grant. This toolkit does
not store your DigitalOcean credentials. See the
[DigitalOcean Privacy Policy](https://www.digitalocean.com/legal/privacy-policy).

## License

MIT — see [LICENSE](LICENSE).
