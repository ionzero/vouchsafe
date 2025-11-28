# Vouchsafe Token Format Specification

**Author:** Jay Kuri  
**Organization:** Ionzero  
**Date:** 2025-11-28    
**Version:** 1.5.0  
**Vouchsafe ID:** urn:vouchsafe:jaykuri.6aublsnyy24dfa6vfil3gxekfcdnmscqvfrcyj4pvvensey6nhla  

## 1. Introduction

This document specifies the Vouchsafe token format, a cryptographically
verifiable data structure for representing trust assertions and attestations.
Vouchsafe tokens are encapsulated in [JSON Web Tokens
(JWT)](https://datatracker.ietf.org/doc/html/rfc7519) and define a
constrained set of claims for expressing portable and decentralized trust
relationships.

A Vouchsafe token enables an issuer to make an explicit, signed statement
regarding the status or authority of another token or identity. Vouchsafe
tokens are designed to function without reliance on centralized authorities or
external resolution mechanisms.

Vouchsafe tokens support the following use cases:

* Issuing a verifiable attestation regarding an identity.
* Delegating trust through chaining of Vouchsafe tokens.
* Revoking a previously issued vouch or all vouches for a subject.
* Permanently destroying an existing identity.

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
claiming `kind: "vch:*"`** and systems that validate them.

---

## 3. Token Structure Overview

Vouchsafe tokens are [JSON Web Tokens
(JWTs)](https://datatracker.ietf.org/doc/html/rfc7519) that include a defined
set of claims and a fixed `kind` value to indicate compliance with the
Vouchsafe format.

All Vouchsafe tokens:

* MUST be signed using an Ed25519 public-key.
* MUST include a defined set of claims used for validation and trust graph construction.
* MUST include a `kind` claim that begins with "vch:" and matches one of the types below.

### 3.1 Base Claims

Every Vouchsafe token MUST include the following claims:

| Claim     | Type   | Description                                                                                                               |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------------------- |
| `iss`     | string | Identifier of the issuer. MUST be a valid [Vouchsafe Identifier](#) derived from the issuer’s public key.                 |
| `iss_key` | string | Base64 of DER encoded public key corresponding to `iss`. MUST match the hash in the URN and be used to verify the JWT signature. |
| `jti`     | string | Unique identifier for the token. MUST be a UUID, lowercase and hyphenated.                               |
| `sub`     | string | The subject of the statement. The interpretation depends on the token type.                                               |
| `iat`     | number | Time the token was issued, in UNIX timestamp format.                                                                      |
| `kind`    | string | Token type indicator. MUST begin with `"vch:"` for Vouchsafe tokens. Valid values are: `vch:attest` `vch:vouch` `vch:revoke` `vch:burn` |

The `exp` (expiration time) claim is OPTIONAL but RECOMMENDED for all tokens 
except revoke tokens.

### 3.2 Claim Semantics

* All Vouchsafe tokens are self-contained. The `iss_key` claim MUST be included
  to allow signature verification without external key resolution.

* The `jti` claim provides a globally unique identifier for the token. It is
  used to support revocation and duplicate detection. Verifiers MUST reject any
token with a malformed or non-UUID `jti`.

* The `sub` claim identifies the subject of the trust assertion. The value and
  semantics of this claim depend on the token’s type.

* The `kind` claim provides a format discriminator. This specification defines
  the values `vch:attest`, `vch:vouch`, `vch:revoke`, `vch:burn`.

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

## 5. Identity and Token ID Guidelines

Vouchsafe tokens use two core fields to establish authorship and uniqueness:

* `iss` — the identity of the token issuer
* `jti` — a globally unique identifier for the token itself

These fields **MUST** be present in all Vouchsafe tokens, and
**SHOULD** be included in any external JWTs that are the subject of a vouch.

> While Vouchsafe tokens define strict requirements to ensure offline
> verification and revocation support, external tokens MAY use alternative
> formats if required by the host system. However, care should be taken as such
> tokens MAY have limited compatibility with Vouchsafe tooling, including trust
> chain verifiers and vouch lookup systems.

### 5.1 `iss`: Identity of the Issuer

* **MUST** be a valid [Vouchsafe Identifier](#) — a URN of the form `urn:vouchsafe:<label>.<base32-hash>`
* **MUST** match the `iss_key` by computing the SHA-256 hash of the extracted raw key bytes and base32-encoding the result
* **MUST** identify the signing principal for the token

### 5.2 `jti`: Token Identifier

The `jti` claim provides a stable, unique reference to a specific token instance.

* **MUST** be a valid UUID (version 4 or 7 recommended)
* **MUST** use lowercase hexadecimal with hyphens (e.g., `"6fa459ea-ee8a-3ca4-894e-db77e160355e"`)
* **MUST NOT** collide with other tokens issued by the same `iss`
* Enables targeted revocation, caching, indexing, and auditability

---

## 6. Vouchsafe Token Types

This section defines the four types of Vouchsafe tokens. Each type specifies a
different form of trust assertion or control operation. All token types use the
same signature and identity validation mechanism as described in Sections 3–5.

The recognized token types are:

1. [Vouchsafe Attestation](#62-vouchsafe-attestation): Declares verifiable claims about an identity.
2. [Vouch for Another Vouchsafe Token](#63-vouch-for-another-vouchsafe-token): Establishes a delegation link by endorsing a prior Vouchsafe token.
3. [Revocation](#64-revocation): Invalidates a previously issued vouch.
4. [Burn](#65-burn): Permanently Invalidates an identity

---

## 6.1 Vouchsafe Attestation

A token that allows a Vouchsafe identity to assert verifiable claims.
Attestations are self-referential and MAY include wrapped external data.
Unlike vouches, attestations do not reference an external token. The statement
exists entirely within the token itself.

### Required Claims

| Claim     | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the attestation.                           |
| `iss_key` | Base64-encoded DER public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this token (UUID).                                                           |
| `sub`     | MUST be equal to the token’s own `jti`. Indicates that the token is self-referential.              |
| `iat`     | UNIX timestamp indicating when the token was issued.                                               |
| `kind`    | MUST be `"vch:attest"`.                                                                                   |

---

### Optional / Recommended Claims

| Claim      | Description                                                                                           |
| ---------- | ----------------------------------------------------------------------------------------------------- |
| `purpose`  | RECOMMENDED. A space-separated list indicating the context or intended use of the attestation.        |
| `exp`      | Optional expiration timestamp. If present, the attestation is not valid after this time.              |
| *(custom)* | Implementations MAY include additional claims representing attributes or declarations being attested. |

---

### Semantics

* The `sub` claim MUST equal the token’s own `jti`. This indicates that the
  attestation refers to itself, rather than a separate subject token.
* The token MAY include arbitrary claims that assert attributes or facts
  declared by the issuer, optionally including wrapped external data.
* The `purpose` claim MAY be used to scope or categorize the statement.
* The `vch_iss` claim MUST NOT be present, as there is no referenced token being vouched for.
* The `vch_sum` claim MUST NOT be present, as there is no referenced token being vouched for.
* The `revokes` claim MUST NOT be present, as there is no referenced token being revoked.
* The `burns` claim MUST NOT be present.

### Example

```json
{
  "iss": "urn:vouchsafe:attestor.2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "sub": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "purpose": "email-verification",
  "email": "user@example.com",
  "iat": 1714605000,
  "kind": "vch:attest"
}
```

## 6.2 Vouch for Another Vouchsafe Token

A token that asserts trust in a previously issued Vouchsafe token. This enables
recursive delegation of trust across multiple identities and permits
construction of verifiable trust graphs. 


### Required Claims

| Claim     | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the vouch.                                 |
| `iss_key` | Base64-encoded DER public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this vouch token (UUID).                                                     |
| `sub`     | MUST be the `jti` of the Vouchsafe token being vouched for.                                        |
| `vch_iss` | MUST match the `iss` of the token being vouched for. MUST be different from `iss`                  |
| `vch_sum` | MUST be the content hash of the full target token (including header and signature).                |
| `iat`     | UNIX timestamp when this token was issued.                                                         |
| `kind`    | MUST be `"vch:vouch"`.                                                                                   |

---

### Optional / Recommended Claims

| Claim     | Description                                                                        |
| --------- | ---------------------------------------------------------------------------------- |
| `purpose` | RECOMMENDED. A space-separated list of permitted uses for the vouched-for token.   |
| `exp`     | Optional expiration timestamp. If present, the vouch is not valid after this time. |

---

### Semantics

* This token references a Vouchsafe token by its `jti`, `vch_iss`, and `vch_sum`.
* The `vch_sum` claim MUST be a content hash of the full target token in its encoded form,
  including all header and payload fields and its signature.
* Trust is scoped to the exact token referenced; if the referenced token is
  modified or reissued with a new `jti`, the vouch does not apply.
* Chained delegation MAY be implemented by issuing vouches that reference other
  Vouchsafe vouch tokens. In such chains, `purpose` filtering applies at each
  link (see Section 9).
* Vouch tokens MUST NOT vouch for another token issued by the same iss.  If a
  revision is needed, the correct approach is to issue a revocation targeting
  the original vouch, followed by a new vouch for the same subject token.
  Alternatively, multiple vouches for the original token may coexist if they
  serve different purposes. 
* Vouch tokens may only vouch for other Vouchsafe tokens. Non-Vouchsafe tokens
  MUST be wrapped inside attestations and cannot appear as `sub` in vouch tokens.

### Example

```json
{
  "iss": "urn:vouchsafe:bob.d4f6p94jwafncxtx63hxvvd1zqt8b2r2",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "4a42c211-8a3c-497d-8a1f-6459fe038301",
  "sub": "c27ae541-d251-4f21-a360-c3b771e3e8a2",
  "vch_iss": "urn:vouchsafe:carol.7b5xkrjq1d3xvsmddq6d58vccu7n07na",
  "vch_sum": "a9b4f6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "purpose": "msg-signing",
  "iat": 1714608000,
  "kind": "vch:vouch"
}
```

---

## **6.3 Revocation**

A revocation token invalidates a previously issued Vouchsafe token.
Revocations apply to:

1. **Individual vouch tokens**, or
2. **All prior vouches issued by the same identity for a specific subject**, or
3. **Individual attestation tokens** (attestations MUST be revoked by explicit `jti`, and `"all"` MUST NOT be used for attestations).

Revocations do not alter the target token itself; they invalidate the trust
assertion or claim previously made by the revoker.

---

### **Required Claims**

| Claim     | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the signer of the revocation.                            |
| `iss_key` | Base64-encoded DER public key that MUST match the `iss` identifier and verify the token signature. |
| `jti`     | Unique identifier for this revocation token (UUID).                                                |
| `sub`     | MUST equal the `jti` of the subject token.                                                         |
| `revokes` | MUST be the `jti` of the specific token being revoked. For vouch tokens only, MAY be `"all"`.      |
| `vch_iss` | MUST equal the `iss` of the subject token.                                                         |
| `vch_sum` | MUST equal the content hash of the subject token.                                                  |
| `iat`     | UNIX timestamp when this token was issued.                                                         |
| `kind`    | MUST be `"vch:revoke"`.                                                                                   |

---

### **Semantics**

* A revocation token invalidates a previously issued Vouchsafe token.
* When `revokes` is a UUID, evaluators MUST treat only that specific token (identified by `jti`, `vch_iss`, and `vch_sum`) as revoked.
* When `revokes` is `"all"`, evaluators MUST revoke **all** vouch tokens previously issued by `iss` for the same subject (`sub`, `vch_iss`, `vch_sum`).
  `"all"` MUST NOT be used when revoking attestation tokens.
* Revocations MUST be content-addressed: `vch_iss` and `vch_sum` MUST match exactly those of the revocation target. 
* Revocation does not modify or overwrite the target token; it removes the revoker's assertion from trust evaluation.
* Revocation tokens MUST NOT include an `exp` claim. Revocations take effect immediately and are permanent.
* If `nbf` is present, the revocation MUST NOT be considered until that time.
* Additional claims SHOULD NOT be added to revocation tokens.
* Revocations MUST NOT target Burn tokens (`vch:burn`) or Revoke tokens (`vch:revoke`)

### **Targeting Rules**

* **Revoking Vouch Tokens**
  For revocations targeting vouch tokens, `sub`, `vch_iss`, and `vch_sum` MUST match exactly the corresponding values in the vouch token being revoked.
  If `revokes` is `"all"`, the revocation applies to all vouch tokens previously issued by the same `iss` with the same `sub`, `vch_iss`, and `vch_sum`.

* **Revoking Attestation Tokens**
  For revocations targeting attestation tokens, `sub`, `vch_iss`, and `vch_sum` MUST match exactly the corresponding values derived from the attestation token being revoked.
  Specifically:

  * `sub` MUST equal the attestation’s `sub` claim (which MUST equal its `jti`),
  * `vch_iss` MUST equal the attestation’s `iss`,
  * `vch_sum` MUST equal the content hash of the attestation token.
  * The value `"all"` MUST NOT be used for `revokes` when revoking attestation tokens.

---

### **Example (Revoking a Vouch)**

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "95f2d4bc-2573-4bb6-8cd1-b6ea4e8e9b71",
  "sub": "c27ae541-d251-4f21-a360-c3b771e3e8a2",
  "revokes": "4a42c211-8a3c-497d-8a1f-6459fe038301",
  "vch_iss": "urn:vouchsafe:carol.7b5xkrjq1d3xvsmddq6d58vccu7n07na",
  "vch_sum": "a9b4f6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "iat": 1714609000,
  "kind": "vch:revoke"
}
```

---

### **Example (Revoking an Attestation)**

```json
{
  "iss": "urn:vouchsafe:bob.88t2kjfzq9p1cew6h5tr6mkq3a4b0m9p",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "6ff51aa3-040a-41a1-89cd-8b19c77260d0",
  "sub": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "revokes": "90e98ed2-2b24-4a22-9985-35a2f23875b4",
  "vch_iss": "urn:vouchsafe:bob.88t2kjfzq9p1cew6h5tr6mkq3a4b0m9p",
  "vch_sum": "c4b9fd1a8f08f30fa3ba5bc4425f79066edb860f478ad98f03ede89ca59f0f14",
  "iat": 1714612000,
  "kind": "vch:revoke"
}
```


## 6.4 Burn - Permanently terminate identity

A **burn token** immediately and permanently invalidates an identity. 
Once a burn token is observed, the issuing identity MUST NOT be considered 
trustworthy for any **future** trust evaluations. Burn tokens do 
**not** retroactively invalidate previous trust decisions. Burn tokens 
MUST invalidate all present and future trust decisions involving 
the relevant `iss`.

A burn token is self-issued: the `burns` MUST be identical to the `iss`.


### **Required Claims**

| Claim     | Description                                                                                                    |
| --------- | -------------------------------------------------------------------------------------------------------------- |
| `iss`     | MUST be a valid Vouchsafe URN. Identifies the identity being burned. MUST match the `sub` claim.               |
| `iss_key` | Base64-encoded DER public key that MUST match the `iss` identifier and verify the token signature.             |
| `jti`     | Unique identifier for this burn token (UUID).                                                                  |
| `sub`     | MUST be equal to the token’s own `jti`. Indicates that the token is self-referential.                          |
| `burns`   | MUST equal `iss`. Indicates that the identity is burning **itself**.                                           |
| `iat`     | MUST be the UNIX timestamp when this token was issued. MUST NOT be in the future relative to evaluation time.  |
| `kind`    | MUST be `"vch:burn"`.                                                                                               |


### **Semantics**

* A burn token permanently retires the issuing identity for all **future** trust evaluations.
* Past decisions regarding vouches, attestations, delegations, and revocations issued by the identity remain historically valid.
* A burn token MUST take effect immediately upon observation, regardless of the value of `iat`.
* Burn tokens MUST NOT retroactively invalidate past trust assertions.
* Burn tokens MUST invalidate future trust assertions involving this iss.
* Burn tokens MUST have a valid `iat`
* If a burn token’s `iat` lies in the future relative to evaluation time, evaluators MUST treat the burn as effective immediately.
* Once a valid burn token is received, all **subsequently evaluated** tokens signed by the same identity MUST be considered invalid, regardless of their `iat` or receipt time.
* Burn tokens MUST have a `burns` field that is identical to `iss`.
* Burn tokens with a `burns` field that is not identical to `iss` are invalid and MUST be discarded.
* Burn tokens MUST NOT include `revokes`, `vch_iss`, `vch_sum`, or any claims associated with vouch revocation.
* Burn tokens MUST NOT include an `exp` claim. Burns are permanent and irrevocable.
* Burn tokens MAY include additional claims but they MUST be ignored by evaluators.

---

### **Example**

```json
{
  "iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "jti": "b0c4d8b1-8e74-4c33-b723-2af0d91ad31e",
  "sub": "b0c4d8b1-8e74-4c33-b723-2af0d91ad31e",
  "burns": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a",
  "iat": 1714702000,
  "kind": "vch:burn"
}
```

## 6.5. Claim Reference Table

This section provides a complete list of all claims defined in the Vouchsafe token format, along with their usage requirements based on token type.

| Claim             | Type   | Description                                                | Required For                      | Notes                                                  |
| ----------------- | ------ | ---------------------------------------------------------- | --------------------------------- | ------------------------------------------------------ |
| `iss`             | string | Issuer identity. MUST be a Vouchsafe URN.                  | All tokens                        | Identity of signer; must match `iss_key` hash          |
| `iss_key`         | string | Base64-encoded DER encoded public key matching `iss`.                  | All tokens                        | Verifies signature and proves identity                 |
| `jti`             | string | Unique token ID (UUID, lowercase w/ hyphens).              | All tokens                        | Target of `revokes`; anchor for trust                  |
| `sub`             | string | Subject of the vouch or attestation.                       | All tokens                        | Interpretation varies by type                          |
| `vch_iss`         | string | Identity of the subject token being vouched for.           | Vouch, Revocation                 |                                                        |
| `vch_sum`         | string | Content hash of the subject token                          | Vouch, Revocation                 | Ensures token integrity                                |
| `revokes`         | string | Target of revocation: either a vouch `jti`, or `"all"`.    | Revocation only                   | Must refer to vouch by same `iss`                      |
| `burns`           | string | MUST equal `iss`           | Burn only                     | Must be same as `iss`, Burns act on the issuer's identity only.                            |
| `iat`             | number | Issued-at timestamp (UNIX).                                | All tokens                        | Used for ordering, expiration, revocation              |
| `kind`            | string | MUST begin with `"vch:"` and be one of the defined types   | All tokens                        | Identifies token format                                |
| `exp`             | number | Expiration timestamp (UNIX).                               | RECOMMENDED (except in revokes)   | After this time, the token is invalid                  |
| `purpose`         | string | Space-separated list of permitted uses.                    | RECOMMENDED (vouch, attestation)  | Used for delegation filtering                          |
| Additional Claims | any    | Arbitrary claims outside the Vouchsafe Specification.      | Optional                          | MAY appear in any token, ignored by Vouchsafe tooling. |

> Note that `exp` is STRONGLY RECOMMENDED in all vouch tokens except revokes and burns.
> Revoke tokens MUST NOT include an `exp` claim.
> Burn tokens MUST NOT include an `exp` claim.

### Usage by Token Type

| Token Type                | `vch_iss` | `vch_sum` | `revokes` | `purpose` | `burns` |
| ------------------------- | --------- | --------- | --------- | --------- | ------- |
| Vouchsafe attestation     |           |           |           | R         |         |
| Vouch for Vouchsafe token | X         | X         |           | R         |         |
| Revocation                | X         | X         | X         |           |         |
| Burn                      |           |           |           |           | X       |

> X = REQUIRED - MUST be included 
> R = RECOMMENDED - MAY be included


## 7. Integrating Non-Vouchsafe Tokens

External (non-Vouchsafe) tokens **MUST NOT** be used as the subject of a vouch.

External data **MAY** be incorporated into a Vouchsafe system only by
**wrapping** the data inside an attestation token.

Wrapped data is treated as opaque and does not participate directly in
Vouchsafe trust evaluation.

### 7.1 General Requirements

1. **External tokens MUST NOT be used as vouch subjects.**
2. **External data MUST be wrapped in an attestation token to appear in a Vouchsafe evaluation set.**
3. **Wrapped external data MUST be treated as opaque by evaluators.**
4. **Evaluators MUST NOT validate or interpret external token signatures, issuers, lifetimes, or internal claims.**
5. **Applications MAY validate external tokens at the application layer.**

### 7.2 Integration Method A: Wrapped External Token (Direct Inclusion)

An attestation token **MAY** wrap an external token directly as an
application-defined claim.

Example pattern:

```json
{
  "iss": "urn:vouchsafe:issuer....",
  "iss_key": "BASE64...",
  "jti": "UUID",
  "sub": "UUID",
  "purpose": "integration",
  "wrapped": {
    "external_token": "<opaque external token>"
  },
  "iat": 1714600000,
  "kind": "vch:attest"
}
```

**Requirements:**

* The wrapped token **MUST** be treated as an opaque, uninterpreted object.
* Only the attestation token's own fields participate in evaluation.


### 7.3 Integration Method B: Wrapped Reference to External Data

An attestation token **MAY** wrap a reference to an external token rather than
the token itself.

Example pattern:

```json
{
  "iss": "urn:vouchsafe:issuer....",
  "iss_key": "BASE64...",
  "jti": "UUID",
  "sub": "UUID",
  "purpose": "integration",
  "wrapped": {
    "external_ref": {
      "id": "opaque-identifier",
      "hash": "sha256-of-external-object",
      "type": "external-format-identifier"
    }
  },
  "iat": 1714600000,
  "kind": "vch:attest"
}
```

**Requirements:**

* The `hash` value **MUST** uniquely identify the referenced object.
* Evaluators **MUST** treat the reference as opaque metadata.
* Evaluators **MUST NOT** dereference or retrieve external content.

---

### 7.4 Migration Recommendation

Systems that rely on external tokens **SHOULD** wrap them in Vouchsafe
attestations whenever integration is required, and **SHOULD** prefer native
Vouchsafe tokens where possible for consistent and complete evaluation
semantics.


## 8. Token Validation Procedure

This section describes the steps a verifier SHOULD follow to validate a
Vouchsafe token (`kind: "vch:*"`). The process includes identity verification,
signature validation, field consistency checks, and — where applicable — vouch
graph resolution and revocation handling.

---

### 8.1 Step-by-Step Verification

#### Step 1: Confirm Required Claims

Ensure the token includes all mandatory claims for its type. At a minimum,
every token MUST include:

* `iss`, `iss_key`, `jti`, `sub`, `iat`, `kind`

Additional required claims depend on the token type (see 
[Section 6.5](#65-claim-reference-table)).

#### Step 2: Verify `kind` is `"vch:*"`

Reject any token where `kind` is missing or not one of the allowed 
types: `"vch:attest"` `"vch:vouch"` `"vch:revoke"` `"vch:burn"`

#### Step 3: Validate `iss` and `iss_key`

* Decode `iss_key` (base64)
* Extract the raw public key bytes from the DER encoding
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

* If the token is an **attestation**, ensure:

  * The token MUST include `kind: "vch:attest"`
  * `sub` MUST equal `jti`
  * No trust chaining is attempted

* If the token is a **vouch**:

  * The token MUST include `kind: "vch:vouch"`
  * Match `vch_iss` and `vch_sum` to the target token

* If the token is a **revocation**, ensure:

  * The token MUST include `kind: "vch:revoke"`
  * `revokes` MUST be present
  * `revokes` MUST be either a valid UUID or `"all"`
  * `sub`, `vch_iss`, and `vch_sum` MUST match the original target of the vouch

* If the token is a **burn**, ensure:

  * The token MUST include `kind: "vch:burn"`
  * `sub` MUST equal `jti`
  * `burns` MUST equal `iss`
  * No trust chaining is attempted

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

In multi-hop delegation chains, Vouchsafe verifiers apply intersectional filtering over all declared `purpose` values. A string is considered valid only if it appears in every token that includes a `purpose` claim. Tokens that omit the `purpose` claim do not constrain the set but also do not expand it.

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

When evaluating a chain of Vouchsafe tokens that includes the `purpose` claim, verifiers MUST apply intersectional filtering. The resulting set of valid `purpose` values is the set of strings that appear in **every** `purpose` claim in the chain.

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

### 10.1 Attestation: Payment Confirmation

**Scenario:**
A payment processor issues a Vouchsafe attestation indicating that a user
completed a successful payment. The attestation includes purpose scoping and
additional application-defined claims.

**Attestation Token Claims:**

```json
{
  "jti": "0a6b2f33-7ac1-4e4b-9b5d-bc5c3a2a91e7",
  "kind": "vch:attest",
  "iss": "urn:vouchsafe:paymentnet.9c2xtrpx26dy44hfcf0dddf5r6ez5pwo",
  "iss_key": "BASE64ENCODEDPUBLICKEY==",
  "sub": "0a6b2f33-7ac1-4e4b-9b5d-bc5c3a2a91e7",
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

### 10.2 Vouch for Another Vouchsafe Token

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
  "vch_sum": "a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "purpose": "msg-signing",
  "iat": 1714602000,
  "kind": "vch:vouch"
}
```

---

### 10.3 Revocation: Specific Token

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
  "vch_sum": "a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530",
  "iat": 1714603000,
  "kind": "vch:revoke"
}
```

---

## 11. Appendices

This section contains supporting information for implementers, including
encoding formats, hash computation, and Vouchsafe URN structure.

---

### 11.1 Hashing and Encoding Standards

Vouchsafe tokens use cryptographic hash functions to bind token
contents. The following conventions apply:

* The hash algorithm used in this specification is **SHA-256** unless otherwise specified.
* Hash outputs MUST be encoded using lowercase, hexadecimal.
* This hexadecimal encoding requirement applies only to `vch_sum` and NOT to the issuer key hash used in Vouchsafe URNs (which use Base32).
* Hash values MUST be expressed using the `vch_sum` claim in the following format:

```
<hexadecimal-encoded-hash>
```

The `<hexadecimal-encoded-hash>` portion MUST be lowercase. 

#### Example

```
a9f4d6c1cc69eac90a63a8df7b0f06cce3ed62ed65031c81ff4c5b8cbd2f5530
```


All compliant implementations MUST use **SHA-256**. Support for alternate hash
algorithms is not defined in this version of the specification and MAY be
introduced in a future revision.

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
4. Confirm that the encoded hash matches the hash portion of the `iss` claim.

If the computed hash does not match the URN, the token MUST be rejected.

---

### 11.3 Supported Algorithms

This specification defines support for the following cryptographic algorithms:

| Algorithm | Purpose | Status                                          |
| --------- | ------- | ----------------------------------------------- |
| Ed25519   | Signing | REQUIRED for signature generation            |
| SHA-256   | Hashing | REQUIRED for `vch_sum` and `iss` URN derivation |

All compliant Vouchsafe tokens MUST support `SHA-256` as the hashing algorithm. Implementations MUST support `Ed25519` for signing and verification.

Support for additional algorithms MAY be introduced in future revisions of this specification. Verifiers MAY reject tokens that use unsupported or deprecated algorithms.

---

## 12. Versioning and Compatibility

This document defines version 1.5 of the Vouchsafe token format. All tokens
conforming to this specification MUST include the claim `kind: "vch:*"` and MUST
adhere to the structural, cryptographic, and validation requirements described
herein.

Future versions of this specification MAY introduce support for additional
claims, algorithms, or token types. Verifiers:

* MUST reject tokens with unknown or unsupported `kind` values.
* MUST ignore unrecognized claims unless explicitly disallowed.
* MUST NOT assume forward compatibility with tokens from future versions unless explicitly specified.

Implementations MAY advertise the specification version(s) they support in
their own metadata or discovery protocols.

---

## 13. Security Considerations

Vouchsafe tokens are designed to enable decentralized, portable, and
cryptographically verifiable trust statements. All tokens are self-contained
and signed using public-key cryptography.

Verifiers MUST reject any token that fails signature verification, has
inconsistent or malformed claim, or contains an `iss` value that does not
match the derived hash of the public key in `iss_key`.

To maintain trust integrity and prevent misuse, implementations SHOULD:

* Enforce strict validation of all required claims by token type.
* Reject tokens with malformed or duplicate `jti` values.
* Limit the depth of delegation chains to prevent resource exhaustion.
* Treat tokens with expired `exp` claim as invalid.
* Reject revocation tokens that conflict or ambiguously overlap with valid vouches.

Vouchsafe tokens cannot form recursive or speculative trust chains. Because
each vouch requires a content hash (`vch_sum`) of its subject token, all
delegation chains are strictly acyclic and must be constructed in dependency
order.

Partial revocation is not supported. To modify trust scopes, issuers MUST
revoke the prior token and issue a new token with revised parameters.

---

## Appendix A. Changelog

### Version 1.5.0 - 2025-11-28

* Revise kind handling to clearly differentiate between different token types
* Remove redundancy and clean up semantic rules where needed

### Version 1.4.0 - 2025-11-07

* Add Burn tokens
* Remove vouch for external tokens in favor of wrapping / referencing external tokens

### Version 1.3.1 - 2025-06-27

* Update spec to clarify that vouching for your own vouch is forbidden.
* Clean up formatting
* Add author URN

### Version 1.3 — 2025-05-10

* Initial public version of the Vouchsafe Token Format Specification.
* Defines core claims, token types, validation rules, URN format, and trust graph semantics.
* Includes examples and implementation guidance for offline verification, revocation, and delegation.

