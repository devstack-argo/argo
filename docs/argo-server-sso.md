# Argo Server SSO

![GA](assets/ga.svg)

> v2.9 and after

## To start Argo Server with SSO.

Firstly, configure the settings [workflow-controller-configmap.yaml](workflow-controller-configmap.yaml) with the correct OAuth 2 values.

Next, create the k8s secrets for holding the OAuth2 `client-id` and `client-secret`. You may refer to the kubernetes documentation on [Managing secrets](https://kubernetes.io/docs/tasks/configmap-secret/). For example by using kubectl with literals:
```
kubectl create secret -n argo generic client-id-secret \
  --from-literal=client-id-key=foo

kubectl create secret -n argo generic client-secret-secret \
  --from-literal=client-secret-key=bar
```

Then, start the Argo Server using the SSO [auth mode](argo-server-auth-mode.md):

```
argo server --auth-mode sso --auth-mode ...
```

## Token Revocation

> v2.12 and after

As of v2.12 we issue a JWE token for users rather than give them the ID token from your OAuth2 provider. This token is opaque and has a longer expiry time (10h by default).

The token encryption key is automatically generated by the Argo Server and stored in a Kubernetes secret name "sso". 

You can revoke all tokens by deleting the encryption key and restarting the Argo Server (so it generates a new key). 

```
kubectl delete secret sso
```

!!! Warning
    The old key will be in the memory the any running Argo Server, and they will therefore accept and user with token encrypted using the old key. Every Argo Server MUST be restarted. 

All users will need to log in again. Sorry.


## SSO RBAC

> v2.12 and after

You can optionally add RBAC to SSO. This allows you to give different users different access levels. Except for `client` auth mode, all users of the Argo Server must ultimately use a service account. So we allow you to define rules that map a user (maybe using their OIDC groups) to a service account by annotating the service account.  

RBAC config is installation-level, so any changes will need to be made by the team that installed Argo. Many complex rules will be burdensome on that team.

Firstly, enable the `rbac:` setting in [workflow-controller-configmap.yaml](workflow-controller-configmap.yaml). You almost certainly want to be able configure RBAC using groups, so add `scopes:` to the SSO settings:

```yaml
sso:
  # ...
  scopes:
   - groups
  rbac:
    enabled: true
```

!!! Note
    Not all OIDC provider support the groups scope. Please speak to your provider about their options.

To configure a service account to be used, annotate it:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  annotations:
    # The rule is an expression used to determine if this service account 
    # should be used. 
    # * `groups` - an array of the OIDC groups
    # * `iss` - the issuer ("argo-server")
    # * `sub` - the subject (typically the username)
    # Must evaluate to a boolean. 
    # If you want an account to be the default to use, this rule can be "true".
    # Details of the expression language are available in
    # https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md.
    workflows.argoproj.io/rbac-rule: "'admin' in groups"
    # The precedence is used to determine which service account to use whe
    # Precedence is an integer. It may be negative. If omitted, it defaults to "0".
    # Numerically higher values have higher precedence (not lower, which maybe 
    # counter-intuitive to you).
    # If two rules match and have the same precedence, then which one used will 
    # be arbitrary.
    workflows.argoproj.io/rbac-rule-precedence: "1"
```


If no rule matches, we deny the user access.

!!! Tip
    You'll probably want to configure a default account to use if no other rule matches, e.g. a read-only account, you can do this as follows:
    
    ```yaml
    metadata:
      name: read-only
      annotations:
        workflows.argoproj.io/rbac-rule: "true"
        workflows.argoproj.io/rbac-rule-precedence: "0"
    ```
    
    The precedence must be the lowest of all your service accounts. 

## Sharing the Argo CD Dex Instance using Oauth2

It is possible to have the Argo Workflows Server use the Argo CD Dex instance for SSO, for instance if you use Okta with SAML which cannot integrate with Argo Workflows directly. In order to make this happen, you will need the following:

- You must be using at least Dex [v2.23.0](https://github.com/dexidp/dex/releases/tag/v2.23.0), because that's when `staticClients[].secretEnv` was added.
- A secret created above with a `client-id` and `client-secret` to be used by both Dex and Argo Workflows Server. It is called `argo-workflows-sso` in this example.
- `--auth-mode=sso` server argument added
- A Dex `staticClients` configured for `argo-workflows-sso`
- The `sso` configuration filled out in Argo Workflows Server to match

What this might look like in your chart configuration:

`argo-cd/values.yaml`:
```yaml
     dex:
       image:
         tag: v2.23.0
       env:
         - name: ARGO_WORKFLOWS_SSO_CLIENT_SECRET
           valueFrom:
             secretKeyRef:
               name: argo-workflows-sso
               key: client-secret
     server:
       config:
         dex.config: |
           staticClients:
           - id: argo-workflows-sso
             name: Argo Workflow
             redirectURIs:
               - https://argo-workflows.mydomain.com/oauth2/callback
             secretEnv: ARGO_WORKFLOWS_SSO_CLIENT_SECRET
```

`argo/values.yaml`:
```yaml
     server:
       extraArgs:
         - --auth-mode=sso
       sso:
         issuer: https://argo-cd.mydomain.com/api/dex
         clientId:
           name: argo-workflows-sso
           key: client-id
         clientSecret:
           name: argo-workflows-sso
           key: client-secret
         redirectUrl: https://argo-workflows.mydomain.com/oauth2/callback
```
