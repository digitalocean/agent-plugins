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

## Arguments

| Argument | Default | Guidance |
| --- | --- | --- |
| `RegistryName` | required | Any registry on the account; the credential works for all of them. |
| `ReadWrite` | `false` | `true` only when pushing. Pull-only consumers (DOKS, App Platform, read replicas of CI) stay read-only. |
| `ExpirySeconds` | unset = never expires | Always set one. Suggested: `3600` interactive push, `604800` (7 d) short-lived CI, `2592000` (30 d) maximum for CI. Non-expiring only on explicit request. |

## Where the output goes

| Consumer | Placement | Notes |
| --- | --- | --- |
| Local `docker` | `~/.docker/config.json` (merge `auths`, do not overwrite other registries) | Prefer `doctl registries login`, which does this for you. |
| GitHub Actions | Repository/organisation secret, then write to `$HOME/.docker/config.json` in a step, or use `digitalocean/action-doctl` + `doctl registries login` with a `DIGITALOCEAN_ACCESS_TOKEN` secret | See the CI/CD how-to for the maintained example. |
| Other CI | Provider secret store; same write-to-file pattern | Never `echo` the secret in logs; mask it if the provider supports masking. |
| Kubernetes (manual) | `kubectl create secret generic <name> --from-file=.dockerconfigjson=<file> --type=kubernetes.io/dockerconfigjson`, then `imagePullSecrets` | `doctl registries kubernetes-manifest` produces the same secret named `registry-<registry>`. |
| Kubernetes (DOKS integration) | None; the control panel manages the secret in every namespace | Preferred for DOKS. |
| App Platform | None; set `image.registry_type: DOCR` in the app spec | No credential handling. |

## Rotation

Credentials with an expiry stop working silently at the deadline; the symptom
is `unauthorized` on pull or push. Record the expiry when you create one and
tell the user. To rotate: mint a new credential, replace it where it is stored,
then verify with a pull. There is no revoke tool for a single credential;
revoking means letting it expire or rotating the underlying API token.
