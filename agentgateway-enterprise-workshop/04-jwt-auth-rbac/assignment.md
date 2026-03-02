---
slug: jwt-auth-rbac
id: 1xtozeho0e1r
type: challenge
title: JWT Authentication & RBAC
teaser: Enforce identity-based access with JWT validation and CEL-based authorization.
notes:
- type: text
  contents: "# \U0001F6E1️ JWT Authentication & RBAC\n\nAPI keys are great for simple
    use cases. But enterprises need **identity-based access control**:\n\n- Validate
    JWT tokens from your identity provider\n- Extract user/team/org claims\n- Enforce
    RBAC rules using CEL expressions\n- Log identity with every request for audit
    trails\n\n AgentGateway supports both inline JWKS and remote JWKS (like Okta,
    Auth0).\n"
tabs:
- id: ywftt61szlme
  title: Terminal
  type: terminal
  hostname: server
- id: 7cgldtuzl8pg
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: xgstfhqxoqks
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: nt3bcqrqzxx5
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# JWT Authentication & RBAC

Let's enforce JWT-based authentication with role-based access control on the gateway.

## Step 1: Clean Up Previous Auth Policy

> **What's happening:** Each challenge demonstrates a different security model, so we remove the API key auth resources from the previous challenge. This ensures a clean slate — the gateway temporarily goes back to allowing unauthenticated access until we apply the new JWT policy. In production, you'd typically transition between auth models more carefully, but for learning purposes this clean-swap approach makes each concept clear.

Remove the API key auth from the previous challenge (if still active):

```bash
kubectl delete enterpriseagentgatewaypolicy api-key-auth -n agentgateway-system 2>/dev/null || true
kubectl delete authconfig apikey-auth -n agentgateway-system 2>/dev/null || true
kubectl delete secret team1-apikey -n agentgateway-system 2>/dev/null || true
```

## Step 2: Ensure the OpenAI Route Exists

> **What's happening:** Same safety check as before — making sure the LLM backend and route are in place so we have traffic to secure.

```bash
kubectl get httproute openai -n agentgateway-system || \
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

## Step 3: Set the JWT Token

> **What's happening:** This JWT was pre-generated with a known RSA private key. When decoded, its payload contains claims like `iss: solo.io`, `org: solo.io`, `team: team-id`, and `llms: { openai: ["gpt-4o"] }`. The `exp` claim is set far in the future so it won't expire during the workshop. In production, tokens would come from your identity provider (Okta, Auth0, Azure AD, etc.) and have short expiration times. The gateway will verify the token's signature against the public key in the JWKS.

We'll use this JWT throughout the challenge. It contains claims: `iss=solo.io`, `org=solo.io`, `team=team-id`:

```bash
export DEV_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw"
```

## Step 4: Create JWT Auth with RBAC Policy

> **What's happening:** This single `EnterpriseAgentgatewayPolicy` does two things. First, `jwtAuthentication` in `Strict` mode means every request *must* have a valid JWT — the gateway verifies the signature using the inline JWKS (the RSA public key), checks the issuer matches `solo.io`, and rejects expired tokens. Second, `authorization.policy.matchExpressions` adds a CEL (Common Expression Language) rule that checks the JWT claims: the request is only allowed if `org == "solo.io"` AND `team == "team-id"`. This is fine-grained RBAC — you can write any boolean expression over JWT claims.

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: agentgateway-jwt-auth
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  traffic:
    jwtAuthentication:
      mode: Strict
      providers:
        - issuer: solo.io
          jwks:
            inline: |
                {
                  "keys": [
                    {
                      "kty": "RSA",
                      "kid": "solo-public-key-001",
                      "n": "vlmc5pb-jYaOq75Y4r91AC2iuS9B0sm6sxzRm3oOG7nIt2F1hHd4AKll2jd6BZg437qvsLdREnbnVrr8kU0drmJNPHL-xbsTz_cQa95GuKb6AI6osAaUAEL3dPjuoqkGNRe1sAJyOi48qtcbV0kPWcwFmCV0-OiqliCms12jrd1PSI_LYiNc3GcutpxY6BiHkbxxNeIuWDxE-i_Obq8EhhGkwha1KVUvLHV-EwD4M_AY8BegGsX-sjoChXOxyueu_ReqWV227I-FTKwMnjwWW0BQkeI6g1w1WqADmtKZ2sLamwGUJgWt4ZgIyhQ-iQfeN1WN2iupTWa5JAsw--CQJw",
                      "e": "AQAB",
                      "use": "sig",
                      "alg": "RS256"
                    }
                  ]
                }
    authorization:
      policy:
        matchExpressions:
          - '(jwt.org == "solo.io") && (jwt.team == "team-id")'
EOF
```

## Step 5: Test Without a JWT (Should Fail — 403)

> **What's happening:** With `mode: Strict`, the gateway requires a valid JWT on every request. Without the `Authorization: Bearer` header, the gateway immediately rejects the request with a `403 Forbidden`. Note this is a 403 (forbidden/unauthorized) rather than a 401 — the gateway's JWT validation runs inline in the data plane, not via the ext-auth-service.

```bash
source /root/.bashrc

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

## Step 6: Test With a Valid JWT (Should Succeed — 200)

> **What's happening:** Now the request includes the JWT as a Bearer token. The gateway verifies the RSA signature using the inline JWKS public key, confirms the issuer is `solo.io`, checks that the token hasn't expired, and then evaluates the CEL expression against the decoded claims. Since `org == "solo.io"` and `team == "team-id"` both match, the request is allowed through to OpenAI. The JWT claims are also captured in access logs and traces for audit purposes.

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "What claims are in my JWT?"}]
  }' | jq .
```

## Step 7: Experiment with RBAC Rules

> **What's happening:** Here you're live-patching the CEL expression to require `org == "internal"` instead of `org == "solo.io"`. Since the JWT's `org` claim is `solo.io`, this expression evaluates to `false`, and the gateway returns `403 Forbidden` even though the JWT itself is valid and properly signed. This demonstrates the power of CEL-based RBAC: you can change access rules dynamically without reissuing tokens or restarting anything. The 3-second sleep gives the control plane time to propagate the change to the data plane.

Patch the policy to require `org=internal` instead:

```bash
kubectl patch enterpriseagentgatewaypolicy agentgateway-jwt-auth -n agentgateway-system \
  --type=merge -p '{"spec":{"traffic":{"authorization":{"policy":{"matchExpressions":["(jwt.org == \"internal\")"]}}}}}'
```

```bash
sleep 3

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "test"}]}'
```

You should see `403 Forbidden`.

## Step 8: Restore the Original RBAC Rule

> **What's happening:** Restoring the original CEL expression so the JWT's claims match again. This round-trip — working → denied → working — demonstrates that RBAC rules can be changed in real time via `kubectl patch` with no downtime or restarts. In production, you'd use GitOps to manage these policies, but the instant feedback loop is the same.

```bash
cat <<'EOF' > /tmp/patch.json
{"spec":{"traffic":{"authorization":{"policy":{"matchExpressions":["(jwt.org == \"solo.io\") && (jwt.team == \"team-id\")"]}}}}}
EOF

kubectl patch enterpriseagentgatewaypolicy agentgateway-jwt-auth -n agentgateway-system \
  --type=merge --patch-file /tmp/patch.json
```

Verify:

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "RBAC restored!"}]
  }' | jq -r '.choices[0].message.content'
```

## Step 9: Check Access Logs

> **What's happening:** The access logs now include a `jwt` field containing the decoded claims from each request's token. This means every LLM request is tagged with the identity of who made it — user, team, org — without any application code changes. In production, this is your audit trail: you can answer "who asked the LLM what, and when?" by querying your log aggregator.

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 4 | jq '{status: ."http.status", jwt: .jwt}' 2>/dev/null || \
kubectl logs deploy/agentgateway -n agentgateway-system --tail 4
```

You can also explore these traces in the **Solo UI** tab.

## ✅ What You've Learned

- `jwtAuthentication` validates tokens using JWKS (inline or remote)
- `authorization.policy.matchExpressions` uses CEL for fine-grained RBAC
- JWT claims are available in access logs and traces for audit

**Next up:** Prompt Enrichment & Guardrails.
