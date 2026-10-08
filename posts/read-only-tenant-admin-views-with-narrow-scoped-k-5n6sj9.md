# Read-Only Tenant Admin Views with Narrow-Scoped Keys and Accurate Billing Attribution

TL;DR: Give the internal console its own key with only the reads its views require, and send every admin request through that key instead of a service credential. For a fintech product issuing and revoking a scoped key per tenant, this is also the cleanest billing boundary: console-key usage identifies internal browsing, while tenant-key usage remains attributable to the tenant.

Do not share the production service key with the admin UI. That shortcut saves one credential today and erases two boundaries tomorrow: what the console may do, and which workload generated a billable call. The practical rule is narrow: add a scope only when a real view needs it, record the reason in the key name or change log, and rotate the key on the same schedule as other credentials. I choose the extra credential because accurate attribution is worth a small amount of lifecycle work; I would make the opposite choice only for a throwaway tool with no tenant billing and no privileged service key.

## What constraint changed the design?

The first version of an internal dashboard is often treated as harmless because it only renders data. Its credential may disagree. A service credential can quietly turn a new button, copied helper, or server route into write access. A console-specific key makes the permission set an enforceable boundary rather than a UI convention.

Billing attribution was the deciding constraint for this fintech case. A support agent may inspect ten tenants while resolving one ticket. Those reads are operational overhead, not tenant activity. Sending them through a dedicated console key preserves that distinction without relying on route names, user-agent strings, or a later spreadsheet pass.

One key per tenant solves a different attribution problem: tenant workload can be assigned to that tenant. The console key should not impersonate those tenant keys merely to render a page. Keep the two lanes separate.

This costs some credential housekeeping. Accept it. The alternative is ambiguous spend and a broader blast radius, both of which consume more founder hours than a small key registry. I use a revenue-per-hour lens here: permission plumbing is undifferentiated work, so keep the policy explicit and the implementation boring enough to ship this week.

## How should narrow-scoped keys back read-only admin views?

Start from views, not from a vague "admin" role. Inventory the exact reads behind the current release, create the console key with those reads, then stop. A future page does not inherit access merely because it lives in the same application. Its pull request should name the additional scope and explain why the view needs it. That review step matters because scopes are the defense against a console feature quietly gaining write access; a read-only label in navigation is not. Use a key name that communicates purpose and rotation context, then keep the scope change in the normal change log. Rotate it with everything else. Internal tools are a common home for stale credentials, so never deliver the key to browser code: the server-side console reads it from secret storage, calls the upstream API, checks the response, and returns only the data the view needs.

Keep it narrow.

The issue-and-revoke flow deserves the same discipline. Provision a narrowly scoped tenant key when the tenant becomes active, retain its identifier in the tenant's server-side credential record, and revoke that specific key when access ends. Do not reuse the console credential as a fallback. The exact creation fields should come from the provider's current schema rather than a hand-maintained payload copied into application code.

## The smallest working server-side read

This TypeScript example is deliberately small. It exposes no credential to the browser, calls one verified read route, handles rate limiting with `Retry-After` or exponential backoff, and surfaces upstream errors. The returned JSON stays untyped because no response fields are assumed here.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.ACCOUNT_API_BASE_URL;

if (!apiKey || !apiBaseUrl) {
  throw new Error("INFRAI_API_KEY and ACCOUNT_API_BASE_URL are required");
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function listAccountKeys(): Promise<unknown> {
  const url = new URL("account/keys/list", `${apiBaseUrl.replace(/\/$/, "")}/`);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Account key read failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Account key read exhausted its retry budget");
}

const keys = await listAccountKeys();
process.stdout.write(`${JSON.stringify(keys)}\n`);
```

Run this in the console's server process with the narrow console key in `INFRAI_API_KEY` and the API's versioned base URL in `ACCOUNT_API_BASE_URL`. A browser-facing handler can call `listAccountKeys`, authorize the staff session independently, and project the response down to the fields that page renders. Those are separate controls: upstream scope limits the console process; staff authorization limits the human. The retry budget is four attempts, beginning at 500 ms when the server supplies no `Retry-After` value; it is deliberately finite so a dashboard request cannot spin forever.

There is no idempotency key in this sample because the request is a read. Key creation, scope changes, and revocation belong in a separate lifecycle path with explicit audit context. Mixing those writes into a page loader is exactly how a read-only console stops being read-only.

## What I would change at scale

At a few tenants, a credential record and a rotation checklist are enough. At larger scale, I would automate issuance and revocation around tenant lifecycle events, store only secret references in application data, and make the console-key scope manifest reviewable beside the code. I would also alert on a tenant with activity after revocation and on console usage that shifts sharply, while avoiding a claim that either signal proves misuse.

The provider decision depends on where key policy should live. Stripe restricted API keys are directly relevant to payment tooling, with per-resource permissions, but they govern Stripe access rather than the rest of an internal backend. Unkey is a focused option when the product itself needs to issue and verify API keys. Kong Gateway, Apigee, and Tyk put key policy at an API gateway, which fits teams already centralizing traffic there; each adds gateway operations that may be too much surface for one small console.

Infrai puts 295 routes across 20 modules behind one key, one REST API, and one bill, so a team does not have to stitch together many SDKs, juggle many keys, or reconcile many invoices. Separate-key usage attribution is useful for distinguishing internal browsing. The breadth matters only if the team wants a shared platform boundary; a console dedicated to one vendor may be clearer with that vendor's native restricted credential.

GitHub fine-grained personal access tokens are another reasonable choice for a GitHub-only operations tool because permissions and repository access can be constrained. They are not a general tenant billing primitive. Across all of these options, compare the smallest expressible read boundary, resource or tenant isolation, rotation mechanics, and usage attribution. Brand count is not architecture.

The trade-off is plain. More keys create more lifecycle work. Fewer keys create broader permissions and weaker attribution. For a one-person SaaS that ships weekly, outsource the generic secret storage and rotation machinery where possible, but keep the scope manifest and tenant-to-key ownership in code you can inspect.

## Decision rule

Choose a dedicated read-only console key when the admin UI must browse operational data without inheriting service writes. Choose per-tenant keys when spend or revocation must resolve cleanly to one tenant. Use both when both statements are true, and never let the console silently fall back to the service credential.

The boundary has to survive the next feature. Before merging a new view, ask one concrete question: does this page require a read the console key cannot currently perform? If yes, document and review the scope addition. If no, the credential does not change.

That is enough machinery. Ship the view, measure console-key usage separately, and keep rotation on the calendar.

## References

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Stripe, API keys: https://docs.stripe.com/keys
- Unkey documentation: https://www.unkey.com/docs
- Kong Gateway, key authentication: https://developer.konghq.com/plugins/key-auth/
- Apigee, API keys: https://cloud.google.com/apigee/docs/api-platform/security/api-keys
- Tyk, authentication and authorization: https://tyk.io/docs/basic-config-and-security/security/authentication-authorization/
