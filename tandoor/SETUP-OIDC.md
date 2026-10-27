# OIDC SSO for Tandoor via Ory Hydra

Wires Tandoor Recipes to the Ory Hydra server for single sign-on using django-allauth's
generic OpenID Connect provider (`allauth.socialaccount.providers.openid_connect`).

- Issuer / discovery: `https://hydra.daemonagent.net/.well-known/openid-configuration`
- Hydra client: `tandoor` (authorization_code + refresh_token, `client_secret_post`,
  scopes `openid email profile offline_access`)
- Redirect URI registered in Hydra (allauth pattern `/accounts/oidc/<provider_id>/login/callback/`
  with `provider_id: hydra`):
  `https://recipes.daemonagent.net/accounts/oidc/hydra/login/callback/`
- Ingress: `https://recipes.daemonagent.net` (TLS terminated by ingress; Tandoor 2.4.2 already
  sets `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')`, so no
  `ACCOUNT_DEFAULT_HTTP_PROTOCOL` env is needed)

## Env vars added (tandoor-deployment.yaml)

| Env var | Source | Purpose |
|---|---|---|
| `SOCIAL_PROVIDERS` | plain value `allauth.socialaccount.providers.openid_connect` | Loads the allauth OpenID Connect provider module (required by Tandoor 2.x) |
| `SOCIALACCOUNT_PROVIDERS` | `secretKeyRef: tandoor-secrets / SOCIALACCOUNT_PROVIDERS` | Full single-line provider JSON (contains the client secret, so the whole JSON lives in the sealed secret) |

The provider JSON (secret shown as `<SECRET>`):

```json
{"openid_connect":{"OAUTH_PKCE_ENABLED":true,"APPS":[{"provider_id":"hydra","name":"Hydra/daemonagent","client_id":"tandoor","secret":"<SECRET>","settings":{"server_url":"https://hydra.daemonagent.net/.well-known/openid-configuration"}}]}}
```

## Seal the client secret (run on a machine with kubectl + kubeseal access)

The client secret is stored on the docker host at
`192.168.12.173:/opt/ory/.env` as `TANDOOR_OIDC_CLIENT_SECRET`. Never print it or
commit it in plain text.

One-liner: fetches the secret over ssh (without echoing it), builds the provider JSON,
and seals it as the `SOCIALACCOUNT_PROVIDERS` key of the `tandoor-secrets` SealedSecret:

```bash
TANDOOR_OIDC_CLIENT_SECRET="$(ssh 192.168.12.173 'sed -n "s/^TANDOOR_OIDC_CLIENT_SECRET=//p" /opt/ory/.env' | tr -d '"')" \
python3 -c 'import json,os; print(json.dumps({"openid_connect":{"OAUTH_PKCE_ENABLED":True,"APPS":[{"provider_id":"hydra","name":"Hydra/daemonagent","client_id":"tandoor","secret":os.environ["TANDOOR_OIDC_CLIENT_SECRET"],"settings":{"server_url":"https://hydra.daemonagent.net/.well-known/openid-configuration"}}]}}), end="")' \
| kubeseal --raw --namespace tandoor --name tandoor-secrets --controller-namespace kube-system --controller-name sealed-secrets \
> /tmp/tandoor-oidc-sealed.txt
```

- The value is passed via environment only (never in argv, never printed).
- `kubeseal --raw` prints base64 ciphertext, ready to paste into `encryptedData`.
- If the ssh extraction returns quoted values, `tr -d '"'` strips surrounding quotes.

Then add one line under `spec.encryptedData` in `sealed-secrets.yaml`
(entry format — paste the file contents as the value):

```yaml
    SOCIALACCOUNT_PROVIDERS: <contents of /tmp/tandoor-oidc-sealed.txt>
```

## Sync order

Both changes go in the same commit: `sealed-secrets.yaml` **must** include the new
`SOCIALACCOUNT_PROVIDERS` key before (or with) the deployment change, otherwise the pod
fails with `CreateContainerConfigError` until the key exists. ArgoCD syncs Secrets
(incl. SealedSecrets) before Deployments, so a single sync of the updated repo is fine.

## Post-sync verification

```bash
# 1. Pod healthy (no CreateContainerConfigError)
kubectl -n tandoor get pods

# 2. SealedSecret synced and secret contains the new key (do NOT print the value)
kubectl -n tandoor get secret tandoor-secrets \
  -o jsonpath='{.data.SOCIALACCOUNT_PROVIDERS}' | base64 -d \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); app=d["openid_connect"]["APPS"][0]; print("json ok, provider_id =", app["provider_id"], ", client_id =", app["client_id"])'
# expected: json ok, provider_id = hydra , client_id = tandoor

# 3. Hydra discovery reachable from the pod
kubectl -n tandoor exec deploy/tandoor -- curl -s -o /dev/null -w '%{http_code}\n' \
  https://hydra.daemonagent.net/.well-known/openid-configuration
# expected: 200

# 4. Login page renders the Hydra button
kubectl -n tandoor exec deploy/tandoor -- curl -s http://localhost/accounts/login/ | grep -io hydra | head -1
```

Then in a browser:

1. Open `https://recipes.daemonagent.net/accounts/login/` — a `Hydra/daemonagent` button must appear.
2. Click it, authenticate against Hydra, and confirm you land back in Tandoor at
   `/accounts/oidc/hydra/login/callback/` and are logged in.
3. As admin, check `/system/` → *Configured Providers* lists the Hydra app.
4. If login fails, check pod logs: `kubectl -n tandoor logs deploy/tandoor | grep -i allauth`.

## Rollback

Revert the commit and re-sync. The extra `SOCIALACCOUNT_PROVIDERS` key in
`tandoor-secrets` is harmless if left in place.
