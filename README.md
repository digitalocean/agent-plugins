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

## Skills

Skills are loaded automatically when a request matches their description.
Each skill carries its own safety rules for the tools it uses.

| Skill | Useful for |
| --- | --- |
| [docr-manage](skills/docr-manage/SKILL.md) | Create, inspect, and delete Container Registry (DOCR) registries; browse repositories, tags, and manifests; view or change the subscription plan |
| [docr-cleanup](skills/docr-cleanup/SKILL.md) | Free registry storage safely: find untagged or stale images, delete tags/manifests, run and monitor garbage collection |
| [docr-access](skills/docr-access/SKILL.md) | Docker credentials, pushing from workstations or CI, pulling from DOKS and App Platform, diagnosing `ImagePullBackOff` and auth errors |

## MCP server

| Server       | Endpoint                           |
| ------------ | ---------------------------------- |
| digitalocean | `https://mcp.digitalocean.com/mcp` |

## Data

Your AI client sends requests and the required resource details to
`mcp.digitalocean.com`, authenticated with your OAuth grant. This toolkit does
not store your DigitalOcean credentials. See the
[DigitalOcean Privacy Policy](https://www.digitalocean.com/legal/privacy-policy).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add skills, including the
`service-job` naming convention and the [skill template](templates/SKILL.md).

## License

MIT — see [LICENSE](LICENSE).
