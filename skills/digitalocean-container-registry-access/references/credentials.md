# DOCR credential shapes and placement

`docr-docker-credentials` returns the raw text of a Docker `config.json`:

```json
{
  "auths": {
    "registry.digitalocean.com": {
      "auth": "<base64 of username:token>"
    }
  }
}
```

The `auth` value is a secret equivalent to an API token with registry access
for the **whole account**. Handle the entire document as a secret.

## Exposure

When an agent calls this tool, the returned document enters the model's
context and the client's tool-call history. Writing it to a file, a CI
secret, or a Kubernetes secret through another tool call records it there as
well. Treat any credential minted through MCP as exposed for its lifetime.
That is why the skill prefers paths the user runs themselves (`doctl
registries login`, control-panel **Download Docker Credentials**, DOKS
integration) and reserves this tool for explicit requests with a short
expiry.

## Arguments

| Argument | Default | Guidance |
| --- | --- | --- |
| `RegistryName` | required | Any registry on the account; the credential works for all of them. |
| `ReadWrite` | `false` | `true` only when pushing. Pull-only consumers (DOKS, App Platform, read replicas of CI) stay read-only. |
| `ExpirySeconds` | unset = never expires | Always set one. Suggested: `3600` interactive push, `604800` (7 d) short-lived CI, `2592000` (30 d) maximum for CI. Non-expiring only on explicit request. |

## How each consumer should get credentials

Preferred paths keep the secret out of the agent session entirely. The
"minted via MCP" column applies only when the user explicitly asks for it.

| Consumer | Preferred (user runs it) | If minted via MCP |
| --- | --- | --- |
| Local `docker` | `doctl registries login` | User merges the output into `~/.docker/config.json` themselves (do not overwrite other registries' `auths`). |
| GitHub Actions | `DIGITALOCEAN_ACCESS_TOKEN` secret + `digitalocean/action-doctl` + `doctl registries login --expiry-seconds 1200` in the job | User pastes the output into a repository secret; the job writes it to `$HOME/.docker/config.json`. Agent does not write it anywhere. |
| Other CI | Same token + `doctl` pattern | User stores it in the provider's secret store; mask it in logs. |
| Kubernetes (manual) | `doctl registries kubernetes-manifest \| kubectl apply -f -` (creates `registry-<registry>`) | User runs `kubectl create secret generic <name> --from-file=.dockerconfigjson=<file> --type=kubernetes.io/dockerconfigjson`, then sets `imagePullSecrets`. |
| Kubernetes (DOKS integration) | Control panel → registry Settings → DigitalOcean Kubernetes integration | Not applicable. |
| App Platform | `image.registry_type: DOCR` in the app spec | Not applicable. |

## Rotation

Credentials with an expiry stop working silently at the deadline; the symptom
is `unauthorized` on pull or push. Record the expiry when you create one and
tell the user. To rotate: mint a new credential, replace it where it is stored,
then verify with a pull. There is no revoke tool for a single credential;
revoking means letting it expire or rotating the underlying API token.
