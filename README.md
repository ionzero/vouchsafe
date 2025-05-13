# Vouchsafe: Self-Verifying Identity Without Accounts, Servers, or Registries

## What Is Vouchsafe? 

Vouchsafe is a new way to represent identity and trust in your system — one
that doesn’t require accounts, logins, or shared databases.

At its core, a **Vouchsafe ID** is just a string like
`urn:vouchsafe:alice.4z2vjf6...` that’s mathematically tied to a public key.
Anyone with the private key can **sign something to prove they control that
identity**, and anyone else can verify it using only the ID and the public key.
No servers, no network calls, no “is this token valid?” lookup.

On top of that, Vouchsafe defines a **token format built on JWT** — called a
*vouch token* — for expressing trust relationships:

- “I trust this key to act on my behalf”
- “Here’s a claim I verified about someone else”
- “That old key isn’t valid anymore”

These tokens work with any JWT library, are easy to verify offline, and can be
passed around between systems — even ones that don’t know each other — with no
shared infrastructure.

**If you’ve used JWTs for sessions, you already know how this works.**
Vouchsafe just gives you a way to **create persistent identities that don't
need a central database to work** and to **pass around trust in a portable,
verifiable way**.

---

## Why Vouchsafe?

Vouchsafe is designed for systems where identity and trust need to move freely
— across services, networks, or devices — without depending on centralized
directories, shared databases, or real-time validation.

It’s ideal for decentralized protocols, cross-platform apps, offline
interactions — or anywhere you need different systems to agree on identity,
capability, and permission. Vouchsafe simplifies these integrations by
replacing custom trust logic with cryptographically verifiable tokens that work
the same way everywhere.

It’s also great for **simplifying traditional systems** — by making token
verification easier, reducing key distribution headaches, and letting you
represent trust relationships in a standard, portable way that is simple and
straightforward to reason about.

### Analogy: The Bank vs The Festival

Most modern identity systems work like a high-security bank. Every time you
want to do anything — open a door, access a file, make a request — someone
calls a central security desk to ask, *“Is this person allowed to do this?”* If
the directory is down or the guard is on break, the system stops.

Vouchsafe is more like a festival. You show your ID once at the gate, buy your
beer tokens, and from then on you just hand over tokens when you need
something. Nobody makes a call. Nobody checks a central list. The tokens are
cryptographically signed and self-validating — they can’t be faked, and they
don’t need to be looked up.

That’s the difference: with Vouchsafe, **trust travels with the user.**

---

## The Portable Trust Layer

Trust is fundamental to most systems — even simple ones. But trust is often
implied, invisible, or tied to infrastructure that’s hard to change.

Vouchsafe tokens give you a **portable trust layer** — one that works across
systems, across keys, and even across time.

- **Delegation** — “I trust this key to act on my behalf to perform this action.”
- **Revocation** — “That key is no longer valid.”
- **Chaining** — “This token trusts another token, and so on.”
- **Interoperability** — No ecosystem, app or company lock-in. Works in any
  JWT-friendly system.

---

## Core Components

### Vouchsafe Identifiers

At the heart of Vouchsafe is a self-verifying identifier:  
```

urn:vouchsafe:<label>.<public-key-hash>

```

Each identifier is cryptographically bound to a public key and can be validated
by anyone, anywhere — without contacting an authority or resolving a document.
These URNs can be used as stable, portable identifiers across systems,
protocols, or applications.

### Vouchsafe Tokens

Vouchsafe tokens are cryptographically signed trust assertions. They can express:

- Attestations (e.g. "this key controls this email")
- Delegations (e.g. "this agent may act on my behalf")
- Revocations (e.g. "that delegation is no longer valid")
- Endorsement (e.g. "I trust this token and its claims.")

Each token includes the issuer’s identity and key, the subject of the trust
statement, and an optional purpose — all verifiable offline using standard
cryptographic tools. They are fully compatible with JWT libraries.

---

## Example Use Cases

- Replace account-based login with cryptographic identity  
- Let services verify identity and permissions without shared infrastructure  
- Allow decentralized systems to share a common model of identity and trust — without a central authority  
- Delegate signing or access rights to an app, agent, or device  
- Layer trust assertions onto existing JWTs (e.g., OAuth or OpenID tokens)  
- Enable peer-to-peer or offline systems to verify credentials  
- Revoke previously granted access without maintaining state  
- Link external systems using a common, verifiable identity format  

---

## Specifications

The following documents define the Vouchsafe model:

- [Vouchsafe Identity Specification](./vouchsafe-identity-specification.md)  
  *Defines the `urn:vouchsafe` format, validation rules, and comparison to other identity models.*

- [Vouchsafe Token Format](./vouchsafe-specification-v1.3.md)  
  *Specifies the `vch` token format, required claims, signature rules, and trust graph semantics.*

*Language libraries and CLI tools coming soon.*

---

## Get Involved

Vouchsafe is in active development, and we’re looking for collaborators,
testers, and implementers.

If you’re building **decentralized applications**, **peer-to-peer systems**, or
systems that care about **identity, trust, and portable credentials**, we’d
love to hear from you.

Questions, ideas, or contributions?  
Reach out at [dev@ionzero.com](mailto:dev@ionzero.com)

## About

Vouchsafe was created by [Jay Kuri](https://github.com/jaykuri) and is
maintained by [Ionzero](https://ionzero.com), a software engineering firm
focused on durable development and secure, scalable systems for decentralized 
and trust-sensitive applications.

---

## License

Vouchsafe is an open system designed for broad adoption and interoperability.

- **Specifications** are published under [Creative Commons Attribution-ShareAlike 4.0 (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/)  
  You are free to share and adapt them, with attribution and the same license.

All parts of the system are free to use, inspect, and extend. We believe trust
infrastructure should be transparent, verifiable, and resistant to capture.

---

© 2025 Jay Kuri / Ionzero.  
For questions, feedback, or integrations, contact [dev@ionzero.com](mailto:dev@ionzero.com)

