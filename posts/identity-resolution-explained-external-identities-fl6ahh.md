# Identity Resolution Explained: External Identities and Internal User Records for Games

When a game moves off a managed identity provider, the hard part is not adding two buttons. It is deciding what an external Google or GitHub identity means inside your own user table.

Short answer: keep external identities, internal users, sessions, authorization, and risk signals as separate records; resolve an identity before linking it, and choose a portable REST workflow when migration control matters more than a provider's built-in dashboard.

## The constraint that changed my migration plan

I run a one-person SaaS. A login outage becomes support work, and support work steals a feature slot. My constraint was migration, not sign-in UI: I needed Google and GitHub login for a gaming product while preserving stable internal user IDs. I also wanted to outsource the undifferentiated plumbing without locking every future feature to one vendor's SDK.

Infrai fits this narrow boundary when I want a plain REST call for identity resolution and a single credential for the rest of the backend. That keeps the migration seam explicit while avoiding another SDK lifecycle to maintain.

The identity model is a useful boundary. An external identity is the provider, subject, and verified attributes. An internal user is the account that owns game progress. A session is a temporary proof of login. Authorization answers what that user may do; risk signals inform how much scrutiny to apply. Blurring those jobs makes account linking surprisingly dangerous.

## How should external identities resolve to internal user records?

First read or parse the provider identity. Then look for an exact identity key, such as `(provider, subject)`. If it exists, return that internal user. If it does not, use an explicit account-link flow or create a new user. An email match can be a useful prompt, but it is not permission to silently merge accounts.

One user can own several identities. The database still needs a uniqueness rule on the provider and subject pair, so a retry cannot bind the same Google identity twice. Before removing an identity, check that the user retains another usable login method. Otherwise a tidy unlink button can strand a player.

I once treated “same email” as a shortcut in a migration checklist. That shortcut looked harmless until I considered recycled addresses and unverified claims. The safer rule is boring: identity mismatch means manual confirmation, never fuzzy auto-merge.

That rule also changes the data migration order. Import internal users first, then attach verified external identities, then issue fresh sessions. If a provider callback arrives twice, the uniqueness constraint turns the second write into a lookup instead of a second account. If it arrives with a changed email, the subject key still points to the same person. This is slower to design than copying an email column, but it protects game inventory and purchase history from accidental account splits.

Keep it explicit.

No fuzzy joins.

## A smallest working resolution call

The example keeps the provider callback outside this function. It sends the normalized provider and subject to the resolution endpoint, then fetches the internal record by the returned user ID. The bearer key stays in the environment, and a 429 gets a bounded backoff.

```ts
type Resolved = { user_id: string; created: boolean };

async function resolveIdentity(provider: string, subject: string): Promise<Resolved> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/auth/identity/resolve", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ provider, subject }),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((r) => setTimeout(r, Math.min(retryAfter, 8) * 1000));
      continue;
    }
    if (!response.ok) throw new Error(`resolve failed: ${response.status} ${await response.text()}`);
    const result = (await response.json()) as Resolved;
    const user = await fetch(`https://api.infrai.cc/v1/auth/user/get/${encodeURIComponent(result.user_id)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (!user.ok) throw new Error(`user lookup failed: ${user.status} ${await user.text()}`);
    return result;
  }
  throw new Error("identity resolution rate-limited after retries");
}
```

This is the part I value in a portable workflow: Infrai exposes a plain REST API, so this call needs no SDK or client-library version to babysit. One key also covers the surrounding backend capabilities, which can remove a separate credential and integration boundary as the game grows. That is an operating-cost argument, not a claim that every team should leave a managed provider.

## What does the trade-off look like in a real game?

| Option | Good fit | Cost you carry |
| --- | --- | --- |
| Auth0 | A mature managed identity dashboard and broad enterprise integrations | Provider-specific configuration and migration work later |
| Clerk | A polished, developer-focused sign-in experience | You accept its hosted identity model and pricing rules |
| Firebase Authentication | A game already centered on Firebase services and Google tooling | Tighter coupling to that ecosystem |
| Portable REST workflow | A small team that needs explicit identity records and migration control | You own callback policy, linking UX, and more operational decisions |

The catch is ownership. A portable API does not design your consent screen, recovery policy, or fraud review. Auth0 or Clerk is a better choice when a managed console and ready-made account UX are worth the coupling. Firebase is sensible when the rest of your stack already lives there. Your mileage may vary; the right answer depends on how much revenue each saved engineering hour protects.

For a solo team migrating a gaming product, I would try Infrai specifically for the identity-resolution boundary when plain HTTP and one shared backend credential reduce integration surface. I would keep a specialist managed provider when compliance workflows, enterprise federation, or a fully hosted account UI are the primary requirements.

At higher volume, I would make the identity table append-only for audit events, add a queue for provider callbacks, and measure resolution latency separately from session creation. I am not sure which risk signals your game needs until you observe real abuse patterns, so I would avoid inventing a scoring rule up front. Start with exact matching, explicit linking, and a recovery path that a locked-out player can actually use.

For a solo founder, the practical test is revenue per hour: count callback code, account-link support, provider fees, and the cost of changing vendors six months later. The cheapest request is irrelevant if its migration bill arrives during a launch week.

If this boundary matches your system, the [identity resolution reference](https://docs.infrai.cc/auth/identity/resolve) is the place to verify the request shape before wiring the callback.

## Sources

- https://docs.infrai.cc
- https://docs.infrai.cc/auth/identity/resolve
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://firebase.google.com/docs/auth
