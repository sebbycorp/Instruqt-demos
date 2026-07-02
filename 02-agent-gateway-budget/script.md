# Video Script — AI Spend Control with Enterprise AgentGateway Budgets

**Presenter:** Sebastian Maniak
**Runtime target:** ~8–10 minutes
**Format:** Talking-head intro, then screen share of the Instruqt lab (Terminal tab)

Convention used below:

- **SAY** — your spoken lines, written to be read naturally
- **[DO]** — what to do on screen while you say it

---

## 1. Cold open / hook (~45 sec)

**[DO]** On camera, or over a title slide.

**SAY:**

> Hey everyone, Sebastian Maniak here from Solo.io.
>
> Quick question: do you actually know what your AI agents spent yesterday? Not how many requests they made — what they *spent*, in dollars.
>
> Because here's the problem with LLM traffic: rate limits count requests, but requests aren't what costs money. Tokens and dollars are. A single request can cost a hundredth of a cent — or five dollars — depending on the model and the payload. Same request count, wildly different bill.
>
> In today's video I'm going to show you a brand-new capability in Enterprise AgentGateway version 2026.6.3 called **Budgets and Dimensions** — native AI spend control, enforced right at the gateway. We'll go from "the gateway is running" to a hard dollar ceiling that returns a clean 429, in four steps.
>
> Let's jump in.

---

## 2. Setting the stage (~30 sec)

**[DO]** Switch to the lab. Show the Terminal tab.

**SAY:**

> Here's my environment. I've got a Kubernetes cluster with Enterprise AgentGateway already installed, an OpenAI backend configured, and a route at `/openai`. The API key lives server-side in a Kubernetes Secret — my clients never touch provider credentials.
>
> The plan for the next few minutes is simple: verify the gateway, teach it what tokens cost, watch the spend in audit mode, and then enforce a hard limit. Catalog, route, audit, block.

---

## 3. Challenge 1 — The gateway is running (~1.5 min)

**[DO]** Run `kubectl get pods -n agentgateway-system`.

**SAY:**

> First, let's see what's actually running. I've got two important pieces here: the **control plane** — that's `enterprise-agentgateway` — which watches Kubernetes for config changes, and the **data plane** — the `agentgateway` proxy — which is what actually handles the LLM traffic. And notice `ext-cache`, that's Redis — it holds the counters that budgets rely on.

**[DO]** Run the `kubectl api-resources` command, then `kubectl explain enterpriseagentgatewaybudget.spec.budgets --recursive | head -20`.

**SAY:**

> And here's the star of this release: a new Custom Resource Definition called `EnterpriseAgentgatewayBudget`. Look at the fields — `limit`, with a unit of USD or Tokens. A `window`, like per day. And `onBudgetExceeded`, which is either **Audit** or **Block**.
>
> That means your AI spend policy is a first-class Kubernetes resource. It lives in version-controlled YAML, right next to the rest of your gateway config. GitOps for your AI bill.

**[DO]** Run the first curl to `/openai`.

**SAY:**

> Let's send a request through the gateway to make sure traffic flows. I'm asking gpt-4o-mini a fitting question — why do tokens, not requests, drive LLM cost.
>
> There's the answer — and more importantly, look at that `usage` block. Input tokens, output tokens. That's the raw material every budget is built on. Right now the gateway sees *tokens*. Next, we teach it *dollars*.

---

## 4. Challenge 2 — Price every request (~2 min)

**[DO]** Apply the `model-cost-catalog` ConfigMap.

**SAY:**

> A budget in USD only works if the gateway can price a request. So step one: a **model cost catalog**. It's just a ConfigMap with per-million-token rates — and these match OpenAI's public pricing. GPT-4o at two-fifty in, ten dollars out. GPT-4o-mini at fifteen cents in, sixty cents out, per million tokens.

**[DO]** Apply the `EnterpriseAgentgatewayParameters`, then patch the Gateway, then run the rollout status command.

**SAY:**

> Now I load that catalog into the data plane using `EnterpriseAgentgatewayParameters` — the `modelCatalog.sources` field points at the ConfigMap — and attach it to the Gateway through `infrastructure.parametersRef`. One quick rollout, and every response that comes through this proxy gets multiplied by those rates. Token usage times catalog rate equals *realized* dollar cost, computed on every single request.

**[DO]** Apply the `budget-demo` HTTPRoute.

**SAY:**

> One more thing before we budget anything: a dedicated route, `/budget-demo`. Same OpenAI backend, its own path. Why? Because budgets attach to routes. When I trip a hard Block limit on this route later, nothing else using `/openai` is affected. That's blast-radius control, and it's the pattern you want in production.

**[DO]** Run the curl to `/budget-demo` with the cost calculation.

**SAY:**

> Let's price one by hand. Send a completion, take the usage, do the math with the catalog rates... and there it is — this request cost a few thousandths of a cent. Sounds like nothing, right? Now multiply that by thousands of agents running twenty-four seven. It's real money. Time to put a budget on it.

---

## 5. Challenge 3 — Audit the spend (~2 min)

**SAY (before running anything):**

> Now — here's the operational wisdom baked into this feature. Rolling out hard spend limits on day one is how you break production agents. The safe pattern is: **audit first, block later**. Meter real spend against a budget, log when it's exceeded, block nothing — until you trust the numbers.

**[DO]** Apply the `EnterpriseAgentgatewayBudget` with the `demo-usd-audit` entry.

**SAY:**

> So here's a budget: five dollars a day, on this route, in **Audit** mode. The gateway meters the realized USD cost of every request against that limit — but exceeding it only gets logged. Nobody gets blocked.

**[DO]** Apply the `EnterpriseAgentgatewayPolicy` with `entBudgetEnforcement`.

**SAY:**

> And notice — the budget alone does nothing. Enforcement is turned on by a *policy*, targeting exactly one route: `budget-demo`. That separation is deliberate. Platform teams own the budgets; app teams own the routes. And the `/openai` route? No policy, no metering, completely untouched.

**[DO]** Run the for-loop sending three requests.

**SAY:**

> Let's send some metered traffic. Three requests... and all three succeed, because an Audit budget never says no. But every one of them was priced against the catalog and counted toward that five-dollar daily limit.

**[DO]** Run the `kubectl get enterpriseagentgatewaybudget` inspect command.

**SAY:**

> The gateway now *knows* what this route spends per day. Next, we make it *care*.

---

## 6. Challenge 4 — Enforce the budget (~2.5 min)

**[DO]** Apply the updated budget with the `demo-token-block` entry.

**SAY:**

> Here's the moment of truth. I'm updating the same budget resource to hold two entries. The audit entry keeps watching dollars. The new one — `demo-token-block` — caps this route at two thousand tokens per day, and this one is set to **Block**. In real life you'd size this from what audit mode measured; I've made it tiny on purpose so we can trip it on camera.
>
> And notice I didn't touch the policy — new budget entries in the namespace are discovered automatically.

**[DO]** Run the burn loop (six requests with status codes).

**SAY:**

> Now let's burn it. Each of these requests asks for a four-hundred-word answer — six hundred plus tokens a pop — and the loop prints the HTTP status each time.
>
> Watch the codes... 200... 200... 200... and *there it is* — **429**. The budget is exhausted, and look at the token count on that blocked request: nothing. The gateway refused *before* calling OpenAI. A blocked request costs exactly zero dollars. That's not an alert after the fact — that's a ceiling.

**[DO]** Run the final two curls hitting `/budget-demo` and `/openai`.

**SAY:**

> And here's my favorite part — proving the blast radius. `/budget-demo` — 429, still blocked. `/openai` — 200, flowing like nothing happened. The hard limit is scoped to exactly the route I targeted, and everything else that pays the bills keeps working.

---

## 7. Wrap-up (~45 sec)

**[DO]** Back on camera, or over a summary slide with the four steps.

**SAY:**

> So that's Budgets in Enterprise AgentGateway 2026.6.3. Four steps: a cost **catalog** so the gateway can price every request. An isolated **route** so enforcement has a clean blast radius. **Audit** mode to measure real spend risk-free. And **Block** mode for a hard ceiling with a clean, machine-readable 429.
>
> Two things to explore from here. First, **Dimensions** — same release — let a single budget scope per user, per team, or per agent, with separate counters for each. And second, the sizing rule: set your Block limits from what Audit actually measured, not from guesses.
>
> If you want to run this yourself, the hands-on lab is linked below. I'm Sebastian Maniak — thanks for watching, and go put a budget on your agents before they put one on you.

---

## B-roll / edit notes

- Overlay the `usage` JSON block when it first appears in Challenge 1 — it's the concept the whole video hangs on.
- In the burn loop, consider zooming the terminal on the 200 → 429 flip; that's the money shot.
- Lower-third suggestions: "Sebastian Maniak — Solo.io" (intro), "EnterpriseAgentgatewayBudget CRD" (Challenge 1), "Audit → Block" (Challenge 3→4).
- Total spoken word count ≈ 1,150 — comfortably 8–10 minutes with terminal pauses.
