# Lead Path Integrity

**System:** public intake path for the 360 Solutions Group site
**Role:** sole designer, implementer, and operator
**Status:** designed, implemented, and tested as an independent operating laboratory. Pre-customer.

---

## The problem

An intake form makes a promise: a person read this and will follow up.

The failure that matters is not a form that throws an error. It is a form that
returns success and delivers nothing. The visitor believes they made contact and
goes quiet. The operator never learns they existed. The loss is invisible from
both sides, and it compounds silently for as long as it goes unnoticed.

I found this exact defect in my own site before launch. Both intake endpoints
validated the payload, generated a receipt ID, returned `200`, and dropped the
submission on the floor. No email, no webhook, no storage.

## The design decision

I rebuilt the intake path to fail closed.

The rule: **never claim receipt the system cannot honor.** If no delivery channel
is configured, or if delivery fails, the endpoint returns `502` and tells the
visitor plainly that the message did not send, with a phone number and email
address to reach a human directly.

A refusal is recoverable. A false confirmation is not.

Delivery runs through two independent channels — an HTTP webhook and a
transactional email provider. Either one succeeding is sufficient; the second is
redundancy rather than a duplicate-lead problem worth optimizing away.

## What the design bought

When the intake path went live, delivery failed in production. Because the system
was built to fail closed, it refused submissions and surfaced the fallback
contact details instead of quietly accepting leads into a void.

The defect was visible on day one instead of being discovered months later
through leads that never arrived.

## Working the failure

The first hypothesis was the credential. It was wrong, and following it cost real
time — several key rotations that each appeared to change nothing.

The evidence corrected it. Reading the platform's runtime logs rather than
retrying the fix showed the provider returning a bare, non-JSON `Forbidden`. That
provider returns structured JSON for genuine authentication failures, so the
response shape pointed at a malformed request rather than a bad key. The actual
cause was a different environment variable — the one used to build the request
URL — which had been overwritten during earlier troubleshooting. The credential
had been valid the whole time.

Confirming it took two direct calls to the provider's API: one that authenticated
successfully, one that sent successfully. Both passed. That isolated the fault to
configuration rather than credentials.

## Instrumenting the human step

The remaining weakness was not code. It was a manual step with no feedback: a
credential pasted into a masked terminal field that accepts any input and reports
nothing.

I replaced it with a scripted path that validates before it writes:

```
captured key length: 415 (should be 50)
```

That single check converted an invisible failure into a visible one. The value in
the clipboard was not the credential at all — it was unrelated text, silently
accepted on earlier attempts because a masked field shows the same row of dots
either way.

Any manual step in a delivery path deserves a validation gate. The cost is one
line. The alternative is failures that look identical to successes.

## Verification

Both endpoints were tested against both states:

| Condition | Expected | Result |
|---|---|---|
| No delivery channel configured | `502` + fallback contact details | Confirmed |
| Channel configured | `200` + full payload received | Confirmed |

Verified additionally: TypeScript compile clean, production build passing across
all 15 routes, and end-to-end delivery confirmed by a live submission arriving in
the destination inbox.

## Operator handoff

The system is documented for the person who has to run it, not just the person
who built it:

- Required environment variables are documented with the consequence of omitting
  each one.
- Delivery failures are logged with the provider's status code and response body,
  so the next failure is diagnosable from logs rather than by guesswork.
- The fallback path is stated in the visitor-facing copy, so no lead depends on
  the operator noticing an outage.

## Honest maturity

This is a small system, and I am describing it accurately.

It is a two-endpoint intake path on a site I designed and operate myself. It has
not run at enterprise scale, has no customer traction behind it, and its
reliability record is days long rather than years. I built it as an independent
operating laboratory to develop and demonstrate method.

What it does show is how I make decisions when a system touches something that
matters: choose the failure mode deliberately, instrument the step humans get
wrong, let evidence overrule the first hypothesis, and verify both the success
and the failure path before calling it done.
