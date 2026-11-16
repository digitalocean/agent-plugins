# Dedicated Inference MCP tool reference

Source of truth: `pkg/registry/dedicated-inference/` in
[digitalocean-labs/mcp-digitalocean](https://github.com/digitalocean-labs/mcp-digitalocean).
If this table and the live `retrieve_schema` output differ, trust the live
schema.

Tools are in the `dedicated-inference` service. Through the hosted
DigitalOcean MCP server invoke them as `dedicated-inference/<tool>`.

Risk column is the tier from **dedicated-inference-manage**: **Read**,
**Confirm**, **Price**, **Secret**, **Destructive**.

`DedicatedInferenceID` is the instance UUID from `dedicated-inference-list`
or `dedicated-inference-get`, not the name.

## Instances

| Tool | Arguments | Risk | Notes |
| --- | --- | --- | --- |
| `dedicated-inference-list` | `Region`, `Name`, `Page`, `PerPage` | Read | Returns `items` plus `meta` when the API sends pagination. Each item has id, name, region, status, VPC UUID, and endpoints. It does not include model deployments; call get for those. |
| `dedicated-inference-get` | `DedicatedInferenceID` (req) | Read | Returns the instance under `dedicated_inference`. Deployment details are on `spec` (`enable_public_endpoint`, `model_deployments` with `model_slug`, `model_id`, `model_provider`, and `accelerators`). Endpoints are `endpoints.public_endpoint_fqdn` and `endpoints.private_endpoint_fqdn`. |
| `dedicated-inference-create` | `Name` (req), `Region` (req), `ModelDeployments` (req), `EnablePublicEndpoint`, `VPCUUID`, `HuggingFaceToken` | Price, and Secret when a Hugging Face token is passed | Sets spec version to 1. `ModelDeployments` entries need `ModelSlug`, `ModelProvider`, and `Accelerators` (`AcceleratorSlug`, `Scale` ≥ 1, `Type`). Returns `dedicated_inference` and a one-time `token`. The token value is not returned again. |
| `dedicated-inference-update` | `DedicatedInferenceID` (req), `ModelDeployments` (required by the handler), `Name`, `Region`, `EnablePublicEndpoint`, `VPCUUID`, `HuggingFaceToken` | Confirm; Price when scale or GPU slug increases; Secret when a Hugging Face token is passed | Empty or missing `ModelDeployments` fails. Resend the current name, region, public-endpoint flag, VPC UUID, and full deployments from get. See the update section in **dedicated-inference-manage**. Omitting `HuggingFaceToken` leaves the existing secret. |
| `dedicated-inference-delete` | `DedicatedInferenceID` (req) | Destructive | Returns `{ "status": "success", "message": "..." }`. The instance, its endpoints, and its access tokens are destroyed. |

## Not on this MCP service

Sizes and prices, GPU-to-model compatibility, accelerator inventory, the CA
certificate, and creating or revoking extra access tokens have no
`dedicated-inference-*` tool. Use the docs and the `doctl dedicated-inference`
commands named in the skills. Do not invent a tool name for them.

Serverless inference, batch inference, the inference router, and model access
keys are different products.
