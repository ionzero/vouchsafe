# Vouchsafe Token Format Specification

**Author:** Jay Kuri 
**Organization:** Ionzero  
**Date:** 2025-05-03  
**Version:** 1.3  
**Vouchsafe ID:** *(to be added: `urn:vouchsafe:jaykuri.<base32-hash>`)*

## 1. Introduction

This document specifies the Vouchsafe token format, a cryptographically
verifiable data structure for representing trust assertions and attestations.
Vouchsafe tokens are based on the [JSON Web Token
(JWT)](https://datatracker.ietf.org/doc/html/rfc7519) format and define a
constrained set of claims for expressing portable and decentralized trust
relationships.

A Vouchsafe token enables an issuer to make an explicit, signed statement
regarding the status or authority of another token or identity. Vouchsafe
tokens are designed to function without reliance on centralized authorities or
external resolution mechanisms.

Vouchsafe tokens support the following use cases:

* Vouching for an arbitrary (non-vouchsafe) JWT.
* Issuing a verifiable attestation regarding an identity.
* Delegating trust through chaining of Vouchsafe tokens.
* Revoking a previously issued vouch or all vouches for a subject.

This specification defines the required and optional claims for each token
type, validation requirements, and the semantics associated with chaining and
revocation.

## 2. Terminology and RFC Conformance

The key words **"MUST"**, **"MUST NOT"**, **"REQUIRED"**, **"SHALL"**, **"SHALL
NOT"**, **"SHOULD"**, **"SHOULD NOT"**, **"RECOMMENDED"**, **"NOT
RECOMMENDED"**, **"MAY"**, and **"OPTIONAL"** in this document are to be
interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) 
([RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119), 
[RFC 8174](https://datatracker.ietf.org/doc/html/rfc8174)) when — and only when —
they appear in **all capital letters** as shown.

These terms indicate the level of requirement placed on implementers of this
specification. Unless otherwise noted, **requirements apply only to tokens
claiming `kind: "vch"`** and systems that validate them.

---

## 3. Token Structure Overview

Vouchsafe tokens are [JSON Web Tokens
(JWTs)](https://datatracker.ietf.org/doc/html/rfc7519) that include a defined
set of claims and a fixed `kind` value to indicate compliance with the
Vouchsafe format.

All Vouchsafe tokens:

* MUST be signed using a supported public-key algorithm (e.g., Ed25519).
* MUST include a defined set of claims used for validation and trust graph construction.
* MUST include a `kind` claim with the exact value `"vch"`.

### 3.1 Base Claims

Every Vouchsafe token MUST include the following claims:

| Claim     | Type   | Description                                                                                                               |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------------------- |
| `iss`     | string | Identifier of the issuer. MUST be a valid [Vouchsafe Identifier](#) derived from the issuer’s public key.                 |
| `iss_key` | string | Base64-encoded public key corresponding to `iss`. MUST match the hash in the URN and be used to verify the JWT signature. |
| `jti`     | string | Unique identifier for the token. MUST be a UUID, lowercase and hyphenated.                               |
| `sub`     | string | The subject of the statement. The interpretation depends on the token type.                                               |
| `iat`     | number | Time the token was issued, in UNIX timestamp format.                                                                      |
| `kind`    | string | Token type indicator. MUST be `"vch"` for Vouchsafe tokens.                                                               |

The `exp` (expiration time) claim is OPTIONAL but RECOMMENDED for all tokens 
except revoke tokens.

### 3.2 Claim Semantics

* All Vouchsafe tokens are self-contained. The `iss_key` claim MUST be included
  to allow signature verification without external key resolution.

* The `jti` claim provides a globally unique identifier for the token. It is
  used to support revocation and duplicate detection. Verifiers MUST reject any
token with a malformed or non-UUID `jti`.

* The `sub` claim identifies the subject of the trust assertion. The value and
  semantics of this field depend on the token’s type.

* The `kind` claim provides a format discriminator. This specification defines
  only the value `"vch"`.

---

### 3.3 Token ID Uniqueness

The combination of `iss` and `jti` MUST be unique across all tokens issued by a
given identity.

Issuers MUST NOT issue more than one token with the same `jti`. Doing so
introduces ambiguity that affects verification, particularly for revocation. If
multiple tokens exist from the same issuer with the same `jti`, a revocation
cannot unambiguously target a specific token.

Verifiers encountering multiple tokens from the same `iss` with the same `jti`
SHOULD treat all such tokens as invalid. The behavior in this case is undefined
and implementation-specific.

If an issuer intends to revise or replace an existing token, it MUST:

* Explicitly revoke the prior token, and
* Issue a new token with a distinct `jti` value.

Tokens are considered immutable. Any attempt to overwrite a previously issued
token without revocation and reissuance will invalidate related trust
statements as vouches rely on `vch_sum` integrity.

---

## 4. Self-Contained Trust Model

Vouchsafe tokens are designed to be portable, independently verifiable, and
free from external dependencies. This is possible because each token carries
everything needed to prove who issued it and verify the statement it contains.

### 4.1 Core Identity Binding

Each Vouchsafe token includes three key elements that work together to
guarantee authenticity:

* **`iss`**: A [Vouchsafe Identifier](#) — a URN derived from the issuer's
  public key. It acts as the identity of the signer.

* **`iss_key`**: The public key corresponding to `iss`. This key is used to
  verify the token's signature and is included directly in the token.

* **JWT signature**: The token is signed using the private key associated with
  `iss_key`. A valid signature proves that the signer controls the private key
  and therefore owns the identity in `iss`.

These elements collectively provide cryptographic proof that the token was
authored by the entity identified by `iss`. Verifiers can validate the identity
and authenticity of the token without relying on any external source of key
material.

A verifier MAY choose to trust a given `iss` identity. If that identity
delegates trust to another party via a Vouchsafe token, the delegation is
explicitly signed and MAY be evaluated according to application-specific
policy.

---

### 4.2 Identity-Centric Trust

Vouchsafe tokens are identity-bound and self-verifying. Each token includes:

* An **identity** (`iss`) represented as a Vouchsafe URN, derived from the public key
* A **public key** (`iss_key`) that can be validated against the identity
* A **digital signature** that proves the signer controls the corresponding private key

Together, these fields ensure that the token was issued by a specific identity,
and that the identity is provably bound to the key used for signing. No
external key registry, trusted directory, or pre-agreed key mapping is
required.

This model allows verifiers to make trust decisions based on identities — not
raw keys — and to verify those identities directly using the contents of the
token itself.

> A verifier may choose to trust `iss`, and thereby accept tokens it has signed.
> If `iss` delegates trust to another party, that delegation is explicitly signed and scoped.

---

## 5. Identity and Token ID Guidelines

Vouchsafe tokens use two core fields to establish authorship and uniqueness:

* `iss` — the identity of the token issuer
* `jti` — a globally unique identifier for the token itself

These fields **MUST** be present in all Vouchsafe tokens (`kind: "vch"`), and
**SHOULD** be included in any external JWTs that are the subject of a vouch.

> While Vouchsafe tokens define strict requirements to ensure offline
> verification and revocation support, external tokens MAY use alternative
> formats if required by the host system. However, care should be taken as such
> tokens MAY have limited compatibility with Vouchsafe tooling, including trust
> chain verifiers and vouch lookup systems.

### 5.1 `iss`: Identity of the Issuer

For Vouchsafe tokens:

* **MUST** be a valid [Vouchsafe Identifier](#) — a URN of the form `urn:vouchsafe:<label>.<base32-hash>`
* **MUST** match the `iss_key` by computing the SHA-256 hash of the decoded key bytes and base32-encoding the result
* **MUST** identify the signing key for the token

For non-Vouchsafe tokens (external JWTs being vouched for):

* **MAY** be a Vouchsafe URN to support trust portability and interoperability
* **MAY** use other formats (e.g., email addresses, URLs, opaque strings) if required by the existing application

### 5.2 `jti`: Token Identifier

The `jti` claim provides a stable, unique reference to a specific token instance.

For Vouchsafe tokens:

* **MUST** be a valid UUID (version 4 or 7 recommended)
* **MUST** use lowercase hexadecimal with hyphens (e.g., `"6fa459ea-ee8a-3ca4-894e-db77e160355e"`)
* **MUST NOT** collide with other tokens issued by the same `iss`
* Enables targeted revocation, caching, indexing, and auditability

For non-Vouchsafe tokens referred to by `sub` in a vouchsafe token:

* **MUST** exist and have a unique value.
* **SHOULD** be a UUID to maximize compatibility with Vouchsafe-based trust tooling
* **MAY** use other unique formats if constrained by the host application

JWTs that do not have a `jti` field can not be the subject of a vouchsafe token.

---

## 6. Vouchsafe Token Types

This section defines the four types of Vouchsafe tokens. Each type specifies a
different form of trust assertion or control operation. All token types use the
same signature and identity validation mechanism as described in Sections 3–5.

The recognized token types are:

1. [Vouch for a Non-Vouchsafe JWT](#61-vouch-for-a-non-vouchsafe-jwt): Asserts trust in an external JWT that is not itself a Vouchsafe token.
2. [Vouchsafe Attestation](#62-vouchsafe-attestation): Declares verifiable claims about an identity.
3. [Vouch for Another Vouchsafe Token](#63-vouch-for-another-vouchsafe-token): Establishes a delegation link by endorsing a prior Vouchsafe token.
4. [Revocation](#64-revocation): Invalidates a previously issued vouch.

---

## 6.1 Vouch for a Non-Vouchsafe JWT

A token that endorses an external JWT, asserting trust in its signer and
contents. This allows Vouchsafe to layer verifiable trust onto existing
systems.

### Required Claims

| Claim     | Description                                                                                    |
| --------- | ---------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the vouch.                             |
| `iss_key` | Base64-encoded public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this vouch token (UUID).                                                 |
| `sub`     | MUST be the `jti` of the external JWT being vouched for.                                       |
| `vch_iss` | MUST match the `iss` of the token being vouched for.                                           |
| `vch_sum` | MUST be the content hash of the full JWT being vouched for, including header and signature.    |
| `iat`     | UNIX timestamp when this token was issued.                                                     |
| `kind`    | MUST be `"vch"`.                                                                               |

---

### Optional / Recommended Claims

| Claim     | Description                                                                                                       |
| --------- | ----------------------------------------------------------------------------------------------------------------- |
| `purpose` | RECOMMENDED. A space-separated list of permitted uses for the vouched-for token (e.g., `msg-signing store-data`). |
| `sub_key` | RECOMMENDED. Base64-encoded public key of the subject, if the external token does not include `iss_key`.          |
| `exp`     | Optional expiration timestamp. If present, the vouch is considered invalid after this time.                       |

---

### Semantics

* This token **does not require** the external JWT to be aware of or compatible
  with Vouchsafe.
* Trust is scoped to the referenced `jti` of the external token, **not the
  identity as a whole**.
* The `vch_sum` ensures that the vouched-for token has not been modified.
* If `sub_key` is provided, verifiers MAY use it to validate the external token
  if it has no embedded key.
* The external JWT **MUST include a valid `jti` field**. If a JWT does not
  contain a `jti`, it **cannot be vouched for**, as there is no stable
  identifier to reference.

### Example

#### External JWT (Token B):

```json
{
  "iss": "user@example.com",
  "jti": "b81b6e6d-c24e-4c9e-b37f-d8a12a5e7609",
  "sub": "user-claim",
  "email": "user@example.com"
}
```

#### Vouch Token (Token A):

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "16db370d-96bb-432c-94b4-19d5808780ec",
  "sub": "b81b6e6d-c24e-4c9e-b37f-d8a12a5e7609",
  "vch_iss": "user@example.com",
  "vch_sum": "sha256:3e9a...e7d5",
  "purpose": "identity-verification",
  "iat": 1714600000,
  "kind": "vch"
}
```

## 6.2 Vouchsafe Attestation

A token that allows a Vouchsafe identity to assert verifiable claims about
another identity or subject. Unlike vouches, attestations do not reference an
external token. The statement exists entirely within the token itself.

### Required Claims

| Claim     | Description                                                                                    |
| --------- | ---------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the attestation.                       |
| `iss_key` | Base64-encoded public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this token (UUID).                                                       |
| `sub`     | MUST be equal to the token’s own `jti`. Indicates that the token is self-referential.          |
| `vch_iss` | MUST identify the subject of the attestation. This is the identity being attested to.          |
| `iat`     | UNIX timestamp indicating when the token was issued.                                           |
| `kind`    | MUST be `"vch"`.                                                                               |

---

### Optional / Recommended Claims

| Claim      | Description                                                                                           |
| ---------- | ----------------------------------------------------------------------------------------------------- |
| `purpose`  | RECOMMENDED. A space-separated list indicating the context or intended use of the attestation.        |
| `exp`      | Optional expiration timestamp. If present, the attestation is not valid after this time.              |
| *(custom)* | Implementations MAY include additional claims representing attributes or declarations being attested. |

---

### Semantics

* The `sub` field MUST equal the token’s own `jti`. This indicates that the
  attestation refers to itself, rather than a separate subject token.
* The `vch_iss` claim identifies the entity being attested to. It MAY be a
  Vouchsafe URN or other identity format depending on the application.
* The token MAY include arbitrary claims that assert attributes or facts about
  the subject.
* The `purpose` claim MAY be used to scope or categorize the statement.
* No `vch_sum` is required, as there is no referenced token being vouched for.

### Example

```json
{
  "iss": "urn:vouchsafe:attestor.2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "sub": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "vch_iss": "urn:vouchsafe:subject.88f13bckjs22a1cwu4txae6cpb9a",
  "purpose": "email-verification",
  "email": "user@example.com",
  "iat": 1714605000,
  "kind": "vch"
}
```


## 6.3 Vouch for Another Vouchsafe Token

A token that asserts trust in a previously issued Vouchsafe token. This enables recursive delegation of trust across multiple identities and permits construction of verifiable trust graphs.

### Required Claims

| Claim     | Description                                                                                    |
| --------- | ---------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the vouch.                             |
| `iss_key` | Base64-encoded public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this vouch token (UUID).                                                 |
| `sub`     | MUST be the `jti` of the Vouchsafe token being vouched for.                                    |
| `vch_iss` | MUST match the `iss` of the token being vouched for.                                           |
| `vch_sum` | MUST be the content hash of the full target token (including header and signature).            |
| `iat`     | UNIX timestamp when this token was issued.                                                     |
| `kind`    | MUST be `"vch"`.                                                                               |

---

### Optional / Recommended Claims

| Claim     | Description                                                                        |
| --------- | ---------------------------------------------------------------------------------- |
| `purpose` | RECOMMENDED. A space-separated list of permitted uses for the vouched-for token.   |
| `exp`     | Optional expiration timestamp. If present, the vouch is not valid after this time. |

---

### Semantics

* This token references a Vouchsafe token by its `jti`, `vch_iss`, and `vch_sum`.
* The `vch_sum` field MUST be a content hash of the full target token,
  including all header and payload fields and its signature.
* Trust is scoped to the exact token referenced; if the referenced token is
  modified or reissued with a new `jti`, the vouch does not apply.
* Chained delegation MAY be implemented by issuing vouches that reference other
  Vouchsafe vouch tokens. In such chains, `purpose` filtering applies at each
  link (see Section 9).

### Example

```json
{
  "iss": "urn:vouchsafe:bob.d4f6p94jwafncxtx63hxvvd1zqt8b2r2",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "4a42c211-8a3c-497d-8a1f-6459fe038301",
  "sub": "c27ae541-d251-4f21-a360-c3b771e3e8a2",
  "vch_iss": "urn:vouchsafe:carol.7b5xkrjq1d3xvsmddq6d58vccu7n07na",
  "vch_sum": "sha256:a9b4f6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "purpose": "msg-signing",
  "iat": 1714608000,
  "kind": "vch"
}
```

---

## 6.4 Revocation

A token that invalidates a previously issued vouch. Revocations apply to
individual vouch tokens or to all vouches issued by the same identity for a
specific subject.

### Required Claims

| Claim     | Description                                                                                               |
| --------- | --------------------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the revocation.                                   |
| `iss_key` | Base64-encoded public key that MUST match the `iss` identifier and verify the token signature.            |
| `jti`     | Unique identifier for this revocation token (UUID).                                                       |
| `sub`     | MUST match the `sub` of the token(s) being revoked.                                                       |
| `revokes` | MUST be either the `jti` of a specific vouch token or the string `"all"` to revoke all vouches for `sub`. |
| `vch_iss` | MUST match the `vch_iss` of the token(s) being revoked.                                                   |
| `vch_sum` | MUST match the `vch_sum` of the token(s) being revoked.                                                   |
| `iat`     | UNIX timestamp when this token was issued.                                                                |
| `kind`    | MUST be `"vch"`.                                                                                          |

---

### Semantics

* A revocation token invalidates a previously issued vouch.
* If `revokes` is a UUID, it MUST match the `jti` of a specific vouch previously issued by the same `iss`.
* If `revokes` is the string `"all"`, all prior vouches issued by `iss` for the same `sub` (with matching `vch_iss` and `vch_sum`) are considered revoked.
* Revocation tokens MUST be evaluated in conjunction with the original vouch token. A revoked vouch MUST be treated as invalid, even if it would otherwise be valid.
* Revocation does not modify the target token itself; it modifies the trust assertion previously made about it.
* Revocation tokens MUST NOT include an exp claim. Revocations take effect immediately upon issuance and are permanent.


### Example

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "95f2d4bc-2573-4bb6-8cd1-b6ea4e8e9b71",
  "sub": "c27ae541-d251-4f21-a360-c3b771e3e8a2",
  "revokes": "4a42c211-8a3c-497d-8a1f-6459fe038301",
  "vch_iss": "urn:vouchsafe:carol.7b5xkrjq1d3xvsmddq6d58vccu7n07na",
  "vch_sum": "sha256:a9b4f6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "iat": 1714609000,
  "kind": "vch"
}
```

---

## 7. Claim Reference Table

This section provides a complete list of all claims defined in the Vouchsafe token format, along with their usage requirements based on token type.

| Claim             | Type   | Description                                                | Required For                      | Notes                                                  |
| ----------------- | ------ | ---------------------------------------------------------- | --------------------------------- | ------------------------------------------------------ |
| `iss`             | string | Issuer identity. MUST be a Vouchsafe URN.                  | All tokens                        | Identity of signer; must match `iss_key` hash          |
| `iss_key`         | string | Base64-encoded public key matching `iss`.                  | All tokens                        | Verifies signature and proves identity                 |
| `jti`             | string | Unique token ID (UUID, lowercase w/ hyphens).              | All tokens                        | Target of `revokes`; anchor for trust                  |
| `sub`             | string | Subject of the vouch or attestation.                       | All tokens                        | Interpretation varies by type                          |
| `vch_iss`         | string | Identity of the subject token being vouched for.           | Vouch, Revocation, Attestation    | Required for all but chained attestations              |
| `vch_sum`         | string | Content hash of the subject token (e.g., `sha256:...`).    | Vouch, Revocation                 | Ensures token integrity                                |
| `revokes`         | string | Target of revocation: either a vouch `jti`, or `"all"`.    | Revocation only                   | Must refer to vouch by same `iss`                      |
| `iat`             | number | Issued-at timestamp (UNIX).                                | All tokens                        | Used for ordering, expiration, revocation              |
| `kind`            | string | MUST be `"vch"` for Vouchsafe tokens.                      | All tokens                        | Identifies token format                                |
| `exp`             | number | Expiration timestamp (UNIX).                               | RECOMMENDED (except in revokes)   | After this time, the token is invalid                  |
| `purpose`         | string | Space-separated list of permitted uses.                    | RECOMMENDED (vouch, attestation)  | Used for delegation filtering                          |
| `sub_key`         | string | Public key of non-Vouchsafe subject (base64).              | Optional for external JWTs        | Enables verification if external token lacks `iss_key` |
| Additional Claims | any    | Arbitrary claims outside the Vouchsafe Specification.      | Optional                          | MAY appear in any token, ignored by Vouchsafe tooling. |

> Note that `exp` is STRONGLY RECOMMENDED in all vouch tokens except revokes.
> Revoke tokens MUST NOT include an `exp` claim.

---

### Usage by Token Type

| Token Type                | `vch_iss` | `vch_sum` | `revokes` | `purpose` | `sub_key`  |
| ------------------------- | --------- | --------- | --------- | --------- | ---------- |
| Vouch for external JWT    | X         | X         |           | R         | R          |
| Vouchsafe attestation     | X         |           |           | R         |            |
| Vouch for Vouchsafe token | X         | X         |           | R         |            |
| Revocation                | X         | X         | X         |           |            |

> X = REQUIRED - MUST be included 
> R = RECOMMENDED - MAY be included

## 8. Token Validation Procedure

This section describes the steps a verifier SHOULD follow to validate a
Vouchsafe token (`kind: "vch"`). The process includes identity verification,
signature validation, field consistency checks, and — where applicable — vouch
graph resolution and revocation handling.

---

### 8.1 Step-by-Step Verification

#### Step 1: Confirm Required Claims

Ensure the token includes all mandatory claims for its type. At a minimum,
every token MUST include:

* `iss`, `iss_key`, `jti`, `sub`, `iat`, `kind`

Additional required claims depend on the token type (see 
[Section 7](#7-claim-reference-table)).

#### Step 2: Verify `kind` is `"vch"`

Reject any token where `kind` is missing or not exactly `"vch"`.

#### Step 3: Validate `iss` and `iss_key`

* Decode `iss_key` (base64)
* Compute the SHA-256 hash of the raw public key bytes
* Base32-encode the result (lowercase, unpadded)
* Confirm the hash matches the `iss` suffix (`urn:vouchsafe:<label>.<hash>`)

If the computed hash does not match the identity, the token MUST be rejected.

#### Step 4: Verify the Signature

* Parse the JWT header and signature
* Use the decoded public key from `iss_key` to verify the signature over the
  token’s header and payload
* If signature verification fails, the token MUST be rejected

#### Step 5: Validate `jti` Claim Format

Ensure `jti` is a valid lowercase, hyphenated UUID (v4 or v7). Reject if
malformed.

#### Step 6: Evaluate Timestamps

* If `exp` is present, it MUST be later than current system time.
* `iat` MUST NOT be later then current system time.

> An implementation MAY allow a small clock skew window when evaluating these
> constraints.

#### Step 7: Enforce Field Consistency

Validate the token's internal logic:

* If the token is a **revocation**, ensure:

  * `revokes` MUST be present
  * `revokes` MUST be either a valid UUID or `"all"`
  * `sub`, `vch_iss`, and `vch_sum` MUST match the original target of the vouch

* If the token is an **attestation**, ensure:

  * `sub` MUST equal `jti`
  * No trust chaining is attempted

* If vouching for a Vouchsafe token:

  * The target token MUST include `kind: "vch"`
  * Match `vch_iss` and `vch_sum` to the target token

* If vouching for a non-Vouchsafe token:

  * The target token `jti` must match `sub`
  * `sub`, `vch_iss`, and `vch_sum` MUST match the original target of the vouch

---

### 8.2 Delegated Trust Evaluation

Implementations that support delegated trust MAY perform trust graph evaluation
to determine whether a subject token is valid within a multi-hop delegation
chain.

To perform this evaluation, verifiers SHOULD:

* Construct the chain of vouch tokens leading to the subject token.
* At each step in the chain:
  * Verify that `vch_sum` and `vch_iss` match the referenced token.
  * Apply intersectional filtering using the `purpose` claims as defined in [Section 9](#9-purpose-and-delegated-trust).
* Evaluate whether any vouches in the chain have been revoked. Revoked vouches MUST be treated as invalid.

Trust graph evaluation MUST be bounded, acyclic, and explicitly
verifier-controlled. Trust is not transitive by default; each delegation step
MUST be explicitly validated and authorized by the verifier.

Section 8.3 is concise and mostly accurate, but to fit RFC tone, we should adjust a few things:

---

### 8.3 Offline Validation

Vouchsafe tokens are self-contained and do not require access to external key
registries or trust databases. Each token includes:

* A cryptographic identity (`iss`) derived from the signing public key.
* A base64-encoded public key (`iss_key`) used to verify the signature.
* A content hash (`vch_sum`) that binds the token to its referenced subject.

If the verifier recognizes and trusts the identity in `iss`, the token MAY be
validated without requiring network access or pre-shared key material.

---

### 9.1 Format and Interpretation

The `purpose` claim, if present, MUST be a space-separated list of purpose strings.

Each string:

* MUST be composed of lowercase ASCII characters.
* MAY include the characters `a–z`, `0–9`, hyphen (`-`), underscore (`_`), and colon (`:`).
* MUST NOT include whitespace or characters outside this set.

The meaning of each purpose string is application-defined. Vouchsafe does not impose semantics on `purpose` values and does not interpret them as permissions, capabilities, roles, or identifiers. Implementations MAY define their own conventions or purpose vocabularies.

In multi-hop delegation chains, Vouchsafe verifiers apply intersectional filtering over all declared `purpose` values. A string is considered valid only if it appears in every token that includes a `purpose` claim. Tokens that omit the `purpose` field do not constrain the set but also do not expand it.

This mechanism enables trust filtering without enforcing a specific purpose model.

#### Examples of valid purpose strings:

```
msg-signing
send-notifications
payment-confirmation
data:read_write
custom-use-case_42
```

---

### 9.2 Purpose Evaluation in Delegated Chains

When evaluating a chain of Vouchsafe tokens that includes the `purpose` claim, verifiers MUST apply intersectional filtering. The resulting set of valid `purpose` values is the set of strings that appear in **every** `purpose` field in the chain.

The following rules apply:

* A token that includes a `purpose` claim MUST explicitly list each permitted value.
* A token that omits the `purpose` claim is considered unconstrained. It does not contribute to the intersection set and does not restrict or expand it.
* A `purpose` value is valid **only if it appears in every token that includes a `purpose` claim**.
* String comparison for `purpose` values is case-sensitive and based on exact byte equality.
* No substring, prefix, or pattern-based matching is permitted.

Implementations MAY define additional local policies for interpreting `purpose` values, but those policies MUST NOT alter the behavior of the intersectional filtering rule as defined above.

---

### 9.3 Revocation Behavior

Revocation applies to the entire vouch token, regardless of any `purpose`
values present. A revoked token is invalid for all purposes.

Partial revocation of specific purposes is not supported. To reduce or modify
the effective `purpose` set, the issuer MUST revoke the original token and
issue a new one with the updated set of allowed values.

---

### 9.4 Example: Purpose Filtering in a Delegation Chain

The following tokens form a delegation chain. Each includes a `purpose` claim
unless noted.

**Token A** — Issued by a user, vouching for an application:

```json
"purpose": "send-notifications store-data"
```

**Token B** — Issued by the application, vouching for a notification agent:

```json
"purpose": "send-notifications"
```

**Token C** — Terminal subject token (no `purpose` claim).

**Result:**
Only `send-notifications` is included in the final effective `purpose` set. The
`store-data` value is excluded because it is not present in Token B.

---

## 10. Example Vouchsafe Tokens

This section provides illustrative, non-normative examples of Vouchsafe tokens used in common scenarios.

---

### 10.1 Vouch for an External JWT

**Scenario:**
A trusted verification service issues a Vouchsafe token referencing an external
JWT. The purpose of the vouch is to assert that the external JWT represents a
verified identity.

**Vouch Token Claims:**

```json
{
  "jti": "27aa21dc-7b76-4e9d-9f1f-6b287e81ac10",
  "kind": "vch",
  "iss": "urn:vouchsafe:verifyd.abc123xyz...",
  "iss_key": "BASE64ENCODEDKEY==",
  "sub": "f197e71d-bfe3-4263-b238-6fe8b9dcb7f0",
  "vch_iss": "user@example.com",
  "vch_sum": "sha256:c8c0f6c60b0bb17a3b16205e1c6c0676019998b7c206372f1128dc37018fa231",
  "purpose": "email-verification",
  "iat": 1714600000
}
```

---

### 10.2 Attestation: Payment Confirmation

**Scenario:**
A payment processor issues a Vouchsafe attestation indicating that a user
completed a successful payment. The attestation includes purpose scoping and
additional application-defined fields.

**Attestation Token Claims:**

```json
{
  "jti": "0a6b2f33-7ac1-4e4b-9b5d-bc5c3a2a91e7",
  "kind": "vch",
  "iss": "urn:vouchsafe:paymentnet.9c2xtrpx26dy44hfcf0dddf5r6ez5pwo",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "sub": "0a6b2f33-7ac1-4e4b-9b5d-bc5c3a2a91e7",
  "vch_iss": "urn:vouchsafe:user.7gk20m4fw8kzzf2dn7a0xns60s64t83p",
  "purpose": "payment-confirmation",
  "amount": "19.95",
  "currency": "USD",
  "paid_for": "enhanced-index-access",
  "transaction_id": "927b395d-bb38-4b64-a91f-9cecb2a1694e",
  "payment_timestamp": 1714598000,
  "iat": 1714601000,
  "exp": 1746137000
}
```

---

### 10.3 Vouch for Another Vouchsafe Token

**Scenario:**
A Vouchsafe token is issued to assert trust in another signed Vouchsafe token. The vouch includes a specific `purpose` to limit delegated authority.

**Vouch Token Claims:**

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "e8b9adfa-6c3e-41e3-a660-031047d422cb",
  "sub": "b7e9c0c2-8f7e-4c90-a4ab-1442a4f432e5",
  "vch_iss": "urn:vouchsafe:agent.4dkf0g2wue68xhzxnslkh9sbk08e4a9b",
  "vch_sum": "sha256:a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "purpose": "msg-signing",
  "iat": 1714602000,
  "kind": "vch"
}
```

---

### 10.4 Revocation: Specific Token

**Scenario:**
A Vouchsafe token is issued to revoke a previously issued vouch by the same
issuer. The revocation is scoped to the specific token identified by its `jti`.

**Revocation Token Claims:**

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "8392c67e-7b4d-4019-a45b-c2605dd067d7",
  "sub": "b7e9c0c2-8f7e-4c90-a4ab-1442a4f432e5",
  "revokes": "e8b9adfa-6c3e-41e3-a660-031047d422cb",
  "vch_iss": "urn:vouchsafe:agent.4dkf0g2wue68xhzxnslkh9sbk08e4a9b",
  "vch_sum": "sha256:a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "iat": 1714603000,
  "kind": "vch"
}
```

---

## 11. Appendices

This section contains supporting information for implementers, including encoding formats, hash computation, and Vouchsafe URN structure.

---

### 11.1 Hashing and Encoding Standards

Vouchsafe tokens use cryptographic hash functions to bind identities and token
contents. The following conventions apply:

* The hash algorithm used in this specification is **SHA-256**.
* Hash outputs MUST be encoded using lowercase, unpadded Base32, as defined in [RFC 4648, Section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6).
* Hash values MUST be expressed using the `vch_sum` claim in the following format:

```
sha256:<base32-encoded-hash>
```

The `<base32-encoded-hash>` portion MUST be lowercase and MUST NOT include
padding.

#### Example

```
sha256:a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530
```

All compliant implementations MUST support `sha256`. Support for alternate hash
algorithms is not defined in this version of the specification and MAY be
introduced in a future revision.

---

### ✅ Revised Section 11.2 – Vouchsafe URN Format (with Reference)

---

### 11.2 Vouchsafe URN Format

Vouchsafe tokens use URNs to represent issuer identities. The syntax and validation rules for these URNs are defined in the [Vouchsafe Identifier Specification](https://example.org/vouchsafe-urn-spec). All compliant implementations MUST validate URNs according to that specification.

For completeness, the basic structure is as follows:

```
urn:vouchsafe:<label>.<base32-key-hash>
```

Where:

* `<label>` is a lowercase, human-readable identifier. It MAY contain the characters `a–z`, `0–9`, and hyphen (`-`). The label MUST NOT exceed 32 characters.
* `<base32-key-hash>` is the lowercase, unpadded Base32 encoding of the SHA-256 hash of the issuer’s raw public key bytes, as defined in [RFC 4648, Section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6).

#### Example

```
urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a
```

To validate a Vouchsafe URN:

1. Base64-decode the `iss_key` claim to obtain the public key bytes.
2. Compute the SHA-256 hash of those bytes.
3. Encode the result using unpadded, lowercase Base32.
4. Confirm that the encoded hash matches the hash portion of the `iss` field.

If the computed hash does not match the URN, the token MUST be rejected.

---

### 11.3 Supported Algorithms

This specification defines support for the following cryptographic algorithms:

| Algorithm | Purpose | Status                                          |
| --------- | ------- | ----------------------------------------------- |
| Ed25519   | Signing | RECOMMENDED for signature generation            |
| SHA-256   | Hashing | REQUIRED for `vch_sum` and `iss` URN derivation |

All compliant Vouchsafe tokens MUST support `SHA-256` as the hashing algorithm. Implementations SHOULD support `Ed25519` for signing and verification.

Support for additional algorithms MAY be introduced in future revisions of this specification. Verifiers MAY reject tokens that use unsupported or deprecated algorithms.

---

## 12. Versioning and Compatibility

This document defines version 1.3 of the Vouchsafe token format. All tokens
conforming to this specification MUST include the claim `kind: "vch"` and MUST
adhere to the structural, cryptographic, and validation requirements described
herein.

Future versions of this specification MAY introduce support for additional
claims, algorithms, or token types. Verifiers:

* MUST reject tokens with unknown or unsupported `kind` values.
* SHOULD ignore unrecognized claims unless explicitly disallowed.
* MUST NOT assume forward compatibility with tokens from future versions unless explicitly specified.

Implementations MAY advertise the specification version(s) they support in
their own metadata or discovery protocols.

---

## 13. Security Considerations

Vouchsafe tokens are designed to enable decentralized, portable, and
cryptographically verifiable trust statements. All tokens are self-contained
and signed using public-key cryptography.

Verifiers MUST reject any token that fails signature verification, has
inconsistent or malformed fields, or contains an `iss` value that does not
match the derived hash of the public key in `iss_key`.

To maintain trust integrity and prevent misuse, implementations SHOULD:

* Enforce strict validation of all required claims by token type.
* Reject tokens with malformed or duplicate `jti` values.
* Limit the depth of delegation chains to prevent resource exhaustion.
* Treat tokens with expired `exp` fields as invalid.
* Reject revocation tokens that conflict or ambiguously overlap with valid vouches.

Vouchsafe tokens cannot form recursive or speculative trust chains. Because
each vouch requires a content hash (`vch_sum`) of its subject token, all
delegation chains are strictly acyclic and must be constructed in dependency
order.

Partial revocation is not supported. To modify trust scopes, issuers MUST
revoke the prior token and issue a new token with revised parameters.

---

## Appendix A. Changelog

### Version 1.3 — 2025-05-10

* Initial public version of the Vouchsafe Token Format Specification.
* Defines core claims, token types, validation rules, URN format, and trust graph semantics.
* Includes examples and implementation guidance for offline verification, revocation, and delegation.

