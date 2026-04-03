# private.id

> Prove who you are without giving yourself away.

## The Problem

Identity verification is broken. Every time you sign up for a rideshare, rent an apartment, or buy a drink — you're asked to hand over your raw license, passport, or personal data. That data gets stored indefinitely, sold, breached, or farmed. You have zero control over where your identity ends up.

Verification should confirm *attributes* — not expose *documents*. A bar needs to know you're over 18. They don't need your home address.

## What private.id Does

private.id is a privacy-proxy verification service. You verify your identity once, receive a cryptographically signed certificate, and use that certificate to prove things about yourself — without ever handing over the underlying data.

- **Zero storage** — no personal data at rest, ever. Documents exist only in-memory during verification, then they're gone.
- **Attribute-based** — certificates confirm facts ("over 18", "valid license") not raw documents.
- **User-controlled** — you choose what to share, and you can revoke it at any time.
- **Asymmetric encryption** — PGP principles applied to identity. You hold the private key. The public verification ID is all that exists on our end.

Third parties validate your certificate through our API. They get a yes/no and the attributes you chose to expose. Nothing else.

We're a conduit, not a vault. You can't leak what you don't have.
