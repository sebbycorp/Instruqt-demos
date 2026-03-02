---
slug: api-key-management
id: el4rppkkfikd
type: challenge
title: API Key Management
teaser: Replace raw LLM provider keys with org-specific vanity API keys.
notes:
- type: text
  contents: "# \U0001F511 API Key Management\n\nYou don't want every developer using
    the raw OpenAI API key. Instead, issue **vanity API keys** per team:\n\n- Teams
    use their own key (e.g., `team1-key`)\n- The gateway validates it and swaps in
    the real provider key\n- You can track usage per team and revoke access instantly\n-
    Additional headers (like `x-org`) get injected for observability\n\nNo more shared
    API keys. No more credential sprawl.\n"
tabs:
- id: mme0weln9q36
  title: Terminal
  type: terminal
  hostname: server
- id: ab0ao9szdsq7
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: uae4gvqsyhoj
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: iyhvi8gq68y9
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# API Key Management

Right now, anyone who can reach the gateway can send requests. Let's fix that by requiring org-specific API keys.

## Step 1: Ensure the OpenAI Route Exists

> **What's happening:** This is a safety check. Since Instruqt challenges build on each other, the OpenAI backend and route from the previous challenge should still be active. If they were cleaned up or something went wrong, this step recreates them so the rest of the challenge works. The route maps `/openai` to the OpenAI backend, which holds the real provider credentials.

The route from the previous challenge should still be active. Verify:

```bash
kubectl get httproute openai -n agentgateway-system
```

If it's not there, recreate it:

```bash
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

## Step 2: Create the API Key, AuthConfig, and Policy

> **What's happening:** Three resources work together here. First, a **Kubernetes Secret** of type `extauth.solo.io/apikey` stores the vanity key (`team1-key`, base64-encoded as `dGVhbTEta2V5`) and a metadata header (`x-org: developers`). Second, an **AuthConfig** tells the ext-auth-service to look for this key in a header called `vanity-auth`, validate it against the secret, and inject the `x-org` header into the upstream request. Third, an **EnterpriseAgentgatewayPolicy** attached to the Gateway tells the data plane to call ext-auth-service for every request. The net effect: clients must present a valid vanity key, and the gateway injects team attribution headers automatically.

This creates:
1. A Kubernetes secret holding the vanity API key (`team1-key`) and an `x-org` header to inject
2. An `AuthConfig` that validates the key and injects the org header
3. An `EnterpriseAgentgatewayPolicy` that enforces auth on the Gateway

```bash
kubectl apply -f- <<EOF
apiVersion: v1
data:
  api-key: dGVhbTEta2V5
  x-org: ZGV2ZWxvcGVycw==
kind: Secret
metadata:
  labels:
    llm-provider: openai
  name: team1-apikey
  namespace: agentgateway-system
type: extauth.solo.io/apikey
---
apiVersion: extauth.solo.io/v1
kind: AuthConfig
metadata:
  name: apikey-auth
  namespace: agentgateway-system
spec:
  configs:
    - apiKeyAuth:
        headerName: vanity-auth
        k8sSecretApikeyStorage:
          apiKeySecretRefs:
            - name: team1-apikey
              namespace: agentgateway-system
        headersFromMetadataEntry:
          x-org:
            name: x-org
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: api-key-auth
  namespace: agentgateway-system
spec:
  targetRefs:
    - name: agentgateway
      group: gateway.networking.k8s.io
      kind: Gateway
  traffic:
    entExtAuth:
      authConfigRef:
        name: apikey-auth
        namespace: agentgateway-system
EOF
```

## Step 3: Test Without an API Key (Should Fail)

> **What's happening:** Now that the auth policy is attached to the Gateway, *every* request must pass authentication. When you send a request without the `vanity-auth` header, the data plane calls the ext-auth-service, which finds no valid key and returns a denial. The gateway translates this into a `401 Unauthorized` HTTP response. This is the "deny by default" security model — the gateway blocks unauthenticated access even though the OpenAI backend is perfectly configured.

```bash
source /root/.bashrc

curl -i "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

You should get a **401 Unauthorized** response. The gateway now requires authentication.

## Step 4: Test With the Vanity API Key (Should Succeed)

> **What's happening:** This time the request includes `vanity-auth: team1-key`. The ext-auth-service looks up this key in the `team1-apikey` secret, finds a match, and tells the gateway to proceed. It also instructs the gateway to inject the `x-org: developers` header into the request before forwarding it to OpenAI. The client never sees or needs the real OpenAI API key — the gateway handles credential injection transparently. If you need to rotate the OpenAI key, you update one secret; if you need to revoke a team's access, you delete their vanity key secret.

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -H "vanity-auth: team1-key" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello from team1!"}]
  }' | jq .
```

The request succeeds. The gateway:
1. Validated `team1-key` against the secret
2. Injected the `x-org: developers` header
3. Swapped in the real OpenAI API key
4. Forwarded the request

## Step 5: Check the Access Logs

> **What's happening:** The access logs now include the injected `x-org` header, giving you per-team attribution for every LLM request. In production, you'd use this to build dashboards showing token usage per team, cost allocation reports, and audit trails. Because the header is injected by the gateway (not the client), it's trustworthy — teams can't spoof another team's identity.

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 5
```

Notice the `x-org` header is captured in the logs — this gives you per-team attribution. You can also check the **Solo UI** tab to view these requests with full tracing details.

## ✅ What You've Learned

- `AuthConfig` (extauth.solo.io) defines API key validation rules
- `EnterpriseAgentgatewayPolicy` enforces auth on the Gateway
- Vanity keys decouple team access from provider credentials
- Header injection enables per-team tracking in logs and metrics

**Next up:** JWT Authentication & RBAC — for fine-grained identity-based access control.
