# Vouchsafe Identity Specification

**Version:** 1.0
**Date:** 2025-05-10
**Author:** Jay Kuri
**Organization:** Ionzero
**URN:** `urn:vouchsafe:jaykuri.<base32-hash>` *(TBD)*

---

## 1. Introduction

This document defines the Vouchsafe identity format and verification model. A Vouchsafe identity is a cryptographically self-verifying Uniform Resource Name (URN) derived from a public key. These identifiers are globally unique, portable, and do not require any registry or resolution infrastructure.

Vouchsafe identities are designed for use in decentralized systems where cryptographic proof of identity is required without reliance on centralized trust anchors or external directories. The identity string (`urn:vouchsafe:<label>.<hash>`) is derived directly from the subject’s public key and can be validated offline by any verifier in possession of the key.

Vouchsafe identifiers are typically used as the issuer (`iss`) field in signed tokens, including [Vouchsafe Tokens](https://example.org/vouchsafe-token-spec). They allow verifiers to confirm both the identity and authenticity of a signed message using only the token contents.

This specification defines:

* The structure of Vouchsafe URNs
* Rules for label formatting and hash encoding
* Procedures for URN validation
* The role of Vouchsafe identities in cryptographic verification
* Security considerations and uniqueness properties

---

## 2. Identifier Format

A Vouchsafe identity is represented as a Uniform Resource Name (URN) with the following general structure:

```
urn:vouchsafe:<label>.<hash>
```

Where:

* `<label>` is an opaque, human-readable identifier.
* `<hash>` is a lowercase, unpadded Base32-encoded hash of the subject’s public key.

### 2.1 Label Format

The label component of the URN:

* MUST be a minimum of 3 characters in length.
* MAY be up to 32 characters in length.
* MAY include the following ASCII characters:
  `a–z`, `A–Z`, `0–9`, hyphen (`-`), underscore (`_`), percent (`%`), plus (`+`)
* MUST NOT contain a period (`.`), which is reserved as a delimiter between the label and the hash.

Labels MAY be URL-encoded to support additional display flexibility.
Applications MAY attempt to decode the label for presentation purposes, but
MUST preserve the original string for all comparisons and URN serialization.

The full URN — including both the label and the hash portion — defines the
identity. Implementations MUST treat different URNs as distinct identities,
even if they share the same key-derived hash. That is,
`urn:vouchsafe:alice.4zxy2d4...` and `urn:vouchsafe:bob.4zxy2d4...` are
separate identities for all purposes, even though they reference the same
public key.

However, only the hash portion is used during cryptographic validation. It MUST
match the hash of the public key in the `iss_key` field when verifying tokens.


### 2.2 Hash Format

The `<hash>` portion is a lowercase, unpadded Base32-encoded SHA-256 digest of the raw public key bytes, as defined in [Section 4](#4-urn-validation). It is used to uniquely bind the URN to the subject's key material.

Implementations MUST use Base32 encoding as defined in [RFC 4648, Section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6), with all output converted to lowercase and padding omitted.

---

### 2.3 Display Recommendations

Although the label component is opaque and has no role in identity
verification, applications MAY implement custom display strategies to improve
usability and recognizability.

The label MAY be URL-encoded by the issuer to support characters that are
otherwise visually ambiguous or reserved in some contexts. However, Vouchsafe
does not require URL encoding, and verifiers MUST NOT rely on decoding behavior
for validation or comparison.

Applications that wish to display the label SHOULD attempt to decode the label
component using standard URL decoding logic. If decoding fails, the raw label
string SHOULD be displayed as-is.

A RECOMMENDED visual representation format is:

```
<label>.<prefix>
```

Where `<label>` is the URL-decoded label (if successful), and `<prefix>` is the
first 4 to 6 characters of the `hash` portion of the URN. This allows users to
visually distinguish between identities while preserving the recognizable
label.

---

## 3. URN Construction

To construct a valid Vouchsafe URN, the issuer MUST derive the identifier from the public key used for signing. The URN format is:

```
urn:vouchsafe:<label>.<base32-hash>
```

Where:

* `<label>` is an application-defined display tag (see [Section 2.1](#21-label-format)).
* `<base32-hash>` is the Base32-encoded output of the hash function.

---

### 3.1 Hash Algorithm

This specification defines the Vouchsafe identity as the combination of a label and a Base32-encoded SHA-256 hash of a public key. 

In this version of the specification:

* SHA-256 is the ONLY supported hash algorithm.
* The `.hash-alg` suffix MUST NOT be present.
* All Vouchsafe URNs MUST use the format:

  ```
  urn:vouchsafe:<label>.<base32-hash>
  ```

To support future extensibility, implementations SHOULD anticipate that
subsequent revisions of this specification MAY introduce alternate hash
algorithms. In such cases, a suffix (e.g., `.sha512`) MAY be appended to the
URN format in a future version. Until such a revision is published, all URNs
should be hashed using SHA-256 and MUST NOT provide a suffix.

### 3.2 Construction Steps

1. **Obtain the public key.**
   The key MUST be in its canonical byte representation as defined by the signature algorithm.

2. **Hash the key.**
   Compute the digest using SHA-256 (or another supported algorithm if explicitly specified).

3. **Encode the hash.**
   Base32-encode the result using the rules defined in [RFC 4648, Section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6):

   * Output MUST be lowercase.
   * Padding MUST be omitted.

4. **Assemble the URN.**
   Construct the URN as follows:

   ```
   urn:vouchsafe:<label>.<base32-hash>
   ```


---

### Example (Default SHA-256)

```
urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a
```

---

## 4. URN Validation

Verifiers MUST validate that a `urn:vouchsafe` identifier is correctly constructed and cryptographically bound to the associated public key. The validation process ensures that the subject in the `iss` claim is authorized to act using the provided `iss_key`.

To validate a Vouchsafe URN:

1. **Extract the hash portion.**

   * Parse the `iss` field and extract the `<base32-hash>` component following the `<label>.` delimiter.

2. **Decode the public key.**

   * Base64-decode the `iss_key` field to obtain the raw public key bytes.

3. **Compute the SHA-256 hash.**

   * Apply the SHA-256 hash function to the raw public key bytes.

4. **Base32-encode the hash.**

   * Encode the hash using unpadded, lowercase Base32, as defined in [RFC 4648, Section 6](https://datatracker.ietf.org/doc/html/rfc4648#section-6).

5. **Compare the computed hash to the URN.**

   * The encoded hash MUST exactly match the `<base32-hash>` portion of the `iss` URN.
   * The comparison MUST be case-sensitive and byte-exact.

If any of these steps fail, the URN MUST be considered invalid, and any associated token or identity assertion MUST be rejected.

---

## 5. Use in Token Systems

In traditional token systems, identities are represented using arbitrary strings — such as email addresses, usernames, or opaque identifiers — whose meaning depends on external configuration or resolution. These identifiers must be interpreted within the context of a trusted directory, database, or mapping to establish authenticity.

Vouchsafe URNs differ in that they are intrinsically bound to public keys. Their structure allows any verifier to confirm the identity’s validity and uniqueness without requiring network access or trusted infrastructure.

This enables identity references that are portable, self-contained, and suitable for expressing authorization, trust, and delegation across system boundaries.

Vouchsafe URNs may appear anywhere an identity needs to be referenced — including as the issuer of a token, the subject of a vouch or attestation, or a participant in a trust graph.


### 5.1 Identity Representation

Vouchsafe URNs MAY be used in any field intended to represent an identity, including but not limited to:

* The `iss` (issuer) claim, identifying the signer of a token
* The `vch_iss` claim, identifying the subject of a vouch or attestation
* Application-specific fields requiring stable, verifiable identifiers

When a Vouchsafe URN appears in the `iss` field, it signifies that the entity
named in the URN has issued the token and has signed it with the corresponding
private key.

Example:

```json
"iss": "urn:vouchsafe:alice.4z2vjf6zjk3j3xkwcu58ftwks61uyd4a"
```

All identity comparisons using Vouchsafe URNs MUST use the full URN string, including the label, as described in [Section 2](#2-identifier-format).

---

### 5.2 Key Binding with `iss_key`

Any token that references a Vouchsafe URN for its signer (e.g., in the `iss` field) MUST also include a corresponding `iss_key` field. This field contains the base64-encoded public key used to:

* Validate the token's signature
* Reconstruct and verify the hash portion of the URN
* Confirm that the signer controls the identity referenced in the token

If the hash derived from `iss_key` does not match the hash portion of the URN in `iss`, the token MUST be rejected.

---

### 5.3 Signature Verification

Verifiers MUST validate the token's digital signature using the decoded `iss_key`. The algorithm used for signature verification MUST match the algorithm declared in the token's header (e.g., `EdDSA` for Ed25519 keys).

Signature verification confirms that the token was generated by an entity in possession of the private key corresponding to the declared identity.

---

## 6. Comparison to Other Identity Models

Vouchsafe identities are one of several approaches to representing unique subjects in distributed systems. This section compares the Vouchsafe model to other common identity formats, focusing on structural and operational differences relevant to implementers.

---

### 6.1 Universally Unique Identifiers (UUIDs)

Universally Unique Identifiers (UUIDs) are commonly used to represent entities across systems. UUIDs are designed to be globally unique, and are often used as primary keys or identity handles within databases or protocols.

However, UUIDs are **opaque and semantically neutral**. They carry no information about the subject they represent and have no inherent connection to a public identifier, verifiable origin, or cryptographic key. Their meaning is entirely contextual and depends on the system in which they are used.

By contrast, Vouchsafe URNs embed meaning directly in the identifier. A Vouchsafe identity includes a human-readable label and a cryptographically derived hash of a public key. This allows the identifier to be verified independently of any system or registry and to retain meaning across organizational boundaries.

While UUIDs represent statistical uniqueness, Vouchsafe URNs represent **provable uniqueness with interpretable structure**, enabling their use in systems that require offline verification and decentralized trust.

---


### 6.2 Centralized Identity Systems

Many identity systems use internal identifiers — such as usernames, email addresses, account IDs, or opaque tokens — that are linked to cryptographic keys within a trusted system or directory. These identifiers often have clear meaning and authorization semantics, but that meaning is defined **entirely within the originating system**.

Outside of that context, the identifiers typically lose their meaning. Even if the identifier is exported, there is no standard mechanism to verify what it represents, who controls it, or how it should be interpreted without consulting the original authority.

By contrast, Vouchsafe identities are **self-contained**. The URN itself is derived from the public key, and the key is included with each signed message. No external lookup is required to determine ownership, and no pre-existing relationship is needed to interpret the identity. This makes Vouchsafe URNs usable in decentralized systems where shared infrastructure or trusted third parties are unavailable or undesired.

While centralized systems are effective within their boundaries, their identifiers are typically **non-portable** and **non-verifiable** outside their origin. Vouchsafe identities are designed to retain their meaning and verification properties across domains.

Although Vouchsafe identities are designed to support decentralized verification, they are also fully compatible with centralized systems. A centralized service MAY assign or link Vouchsafe URNs to internal user records, supplementing or replacing traditional opaque identifiers. This allows the system to retain its existing authorization and storage model while gaining the ability to make use of externally verifiable, cryptographically bound identity claims. In such cases, the Vouchsafe identity can be used both within the originating system and externally, without loss of meaning or the need for out-of-band trust configuration.

---

### 6.3 Decentralized Identifiers (DIDs)

Decentralized Identifiers (DIDs) are a specification for portable, self-managed identifiers. A DID refers to a document — the **DID Document** — which describes how to resolve the identifier to key material, service endpoints, or metadata. The DID Document is typically retrieved via a resolver or registry using a method-specific protocol.

DIDs aim to support flexible identity representations that can include key rotation, delegation, service discovery, and interoperability between systems. The model supports both centralized and decentralized registries and often depends on external infrastructure to retrieve and interpret DID Documents.

By contrast, a Vouchsafe URN is a **self-contained identifier** that does not require resolution. It includes a hash of a public key, and the key is expected to be provided alongside any signed message or token. Identity ownership is proven directly through signature verification and hash comparison, without network access or method-specific logic.

While DIDs are optimized for extensibility and registry-driven identity systems, Vouchsafe identities are optimized for **offline validation, minimal infrastructure, and cryptographic determinism**. Both models are compatible with decentralized systems, but represent different tradeoffs between flexibility and simplicity.

While both DIDs and Vouchsafe URNs aim to enable decentralized identity, they differ in their approach to meaning and resolution. The DID model is based on **retrieval**: the identifier refers to an external document that must be fetched and interpreted to determine the key material and metadata associated with the identity. In contrast, the Vouchsafe model is based on **provisioning**: the identity includes a cryptographic binding to a public key, and any associated metadata or trust statements are supplied explicitly by the subject or referring party.

This allows Vouchsafe identities to be used as globally verifiable anchors for application-defined meaning, without relying on external infrastructure, resolvers, or discovery protocols. Verifiers validate the identity based on the content provided, rather than querying a third party for additional context.

---

### 6.4 Summary

Vouchsafe URNs differ from other identity systems in that they are **self-validating, portable, and cryptographically meaningful** without requiring external infrastructure. Unlike UUIDs or centralized identifiers, they can be verified offline and carry semantic structure. Unlike DIDs, they do not rely on resolver methods or registries to convey identity meaning. This design enables identities that can be freely distributed, reasoned about, and anchored to trust relationships — all using verifiable data provided at the point of interaction.

---

## 7. Security Considerations

Vouchsafe identities are designed to be self-contained, collision-resistant, and independently verifiable. Their security relies on well-established cryptographic primitives and conservative assumptions about verifier behavior.

### 7.1 Key Binding and Verification

A Vouchsafe URN is cryptographically bound to a specific public key. Verifiers MUST validate that the hash portion of the URN matches the hash of the provided key. If the hash does not match, the identity MUST be considered invalid.

Verification of ownership requires a valid digital signature made using the corresponding private key. An attacker cannot impersonate a Vouchsafe identity without access to the private key associated with the public key encoded in the URN.

### 7.2 Hash Collision Resistance

Th  hash algorithm used in Vouchsafe identities is SHA-256. This algorithm is resistant to preimage, second preimage, and collision attacks under current cryptographic assumptions. As such, two different keys cannot be constructed to produce the same URN hash, and an attacker cannot reverse a URN to obtain the corresponding key.

If future vulnerabilities are discovered in SHA-256, this specification may be revised to support additional hash algorithms. Until such time, implementations MUST reject any identity containing an unsupported or unrecognized hash suffix.

### 7.3 Identity Spoofing and Label Ambiguity

The `<label>` component of the URN is not cryptographically bound to the public
key and MUST NOT be used alone to make security decisions. Labels are included
for display and disambiguation purposes only. Systems MUST validate the hash
portion against the key and MUST NOT rely on the label to determine identity or
trustworthiness.

Verifiers SHOULD treat two URNs with the same hash but different labels as
distinct identities, as described in [Section 2.1](#21-label-format).

### 7.4 Key Reuse and Identity Separation

If the same key material is used to construct multiple URNs with different
labels, those URNs are considered separate identities. Systems that allow or
enforce key reuse across multiple identities MUST ensure that trust delegation,
metadata interpretation, and revocation logic operate on full URNs, not just on
key hashes.

### 7.5 Offline Verification and Attack Surface

Vouchsafe identities are intended for offline verification. All data required
to validate the identity must be provided at the time of evaluation. Because no
external resolution step is involved, the attack surface is limited to the
validity of the key, the strength of the hash function, and the correctness of
the signature.

This model reduces dependence on trusted infrastructure but requires that
applications handle trust policy and revocation at the application level.

## Appendix A. Changelog

### Version 1.0 — 2025-05-10

* Initial public version of the Vouchsafe Identity Specification.
* Defines the `urn:vouchsafe` format, construction and validation procedures.
* Describes usage in token systems and comparisons with UUIDs, centralized identity models, and DIDs.
* Includes security considerations for verification, hash integrity, and offline trust.

