# Example

What the three-file context system looks like on a real project after a few months of use.

The project is fictional: **relay**, a TypeScript service that ingests webhooks from Stripe, GitHub, and Shopify, stores them, and fans them out to email, Slack, and SMS. Mid-sized, single node, a handful of hard calls behind it — the kind of project where you forget in June why you ruled something out in February.

| File | What it shows |
|------|---------------|
| `state.md` | A mid-work snapshot, written by preserve at the end of a session. Nothing vague in it. |
| `decisions.md` | Nine months of nothing, then eight entries. One already flipped to `superseded`. |
| `audit-output.md` | A transcript of one audit run over those decisions — what held, what the code broke, and what it refused to guess at. |

`scratch.md` isn't here because it's never worth keeping — preserve harvests what matters into `state.md` and wipes the rest.

The dates and commit hashes are invented. The drift is not: two of these decisions were quietly violated by ordinary-looking commits, which is the whole reason audit exists.
