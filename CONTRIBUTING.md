# Contributing

Thanks for helping improve DigitalOcean's agent skills. This guide covers how
to add and update **skills**.

## Guidelines for adding skills

When contributing new skills to this repository, follow these guidelines:

1. **Skill naming convention** — Use the `service-job` format for skill folder
   names, where:
   - `service` represents the DigitalOcean product MCP (e.g., `droplet`,
     `database`, `apps`, `docr`).
   - `job` represents the specific workflow or task the skill covers (e.g.,
     `troubleshoot`).

2. **Keep skills simple** — Add only essential skills covering commonly used
   tasks and workflows. Avoid creating skills for rarely used or overly specific
   scenarios.

3. **Avoid sensitive information** — Do not include credentials, secrets, API
   tokens, customer data, consumer information, or internal/confidential
   DigitalOcean information in skills. Use generic examples and placeholder
   values wherever necessary.

4. **Follow standard guidelines** — Follow the [Agent Skills specification and
   authoring guidelines](https://agentskills.io/home) when creating or updating
   skills.

5. **Understand the DO MCP tool discovery flow** — Skills must be designed to
   work with the DigitalOcean MCP's three-step tool discovery and invocation
   flow:
   - `find_tools` — discover relevant tools across product MCPs based on the
     user's query.
   - `retrieve_schema` — retrieve the input and output schemas for the selected
     tool to understand its required parameters and expected response.
   - `invoke_tool` — invoke the selected tool with the required input
     parameters. Use output field projection when appropriate to return only the
     relevant fields.

   Skills should guide agents on when and how to use this flow to accomplish
   specific tasks. Do not assume direct access to individual product MCP tools or
   hardcode tool schemas and parameters that should be discovered dynamically.

6. **Safety guidelines** — Include relevant safety instructions in skills
   wherever necessary, especially for destructive, sensitive, or cost-incurring
   operations.

## Skill template

Start from [`templates/SKILL.md`](templates/SKILL.md): copy it into a new
`skills/<service>-<job>/` folder and fill it in. Every `SKILL.md` begins with
YAML frontmatter in this format:

```yaml
---
name: <service>-<job>
description: >-
  <One or two sentences. State what the skill does and the DigitalOcean product
  it covers, then "Use when ..." with the words, errors, and intents that should
  trigger it.>
---
```

- `name` must match the folder name and use the `service-job` convention.
- `description` is what the agent matches against to auto-load the skill — make
  it specific and include trigger phrases.
