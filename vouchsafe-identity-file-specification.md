# Vouchsafe Identity File Specification

**Version:** 1.0.0  
**Status:** Draft Specification  
**Author:** Jay Kuri  
**Organization:** Ionzero

## 1. Introduction

This document specifies the Vouchsafe Identity File format.

A Vouchsafe Identity File contains the cryptographic key material for a Vouchsafe identity.

The Vouchsafe Identity Specification defines the identity. The Identity File does not define the identity. A Vouchsafe identity is a URN that has a cryptographic relation to an Ed25519 public key.

Use the Identity File to store the key material that is necessary to use the identity.

An Identity File contains one of these private-key forms:

* An unencrypted private key.
* A passphrase-encrypted private key.

The encrypted format has these design requirements:

* The format must be compact.
* The format must be easy to implement in different programming languages.
* The JSON must not show cryptographic parameters as configuration options.
* The format must permit future KDF and cipher support without changing the outer JSON structure.
* The private-key data must use the same DER format in encrypted and unencrypted Identity Files.
* The format must use established cryptographic algorithms and must not require novel cryptographic primitives.

Applications SHOULD use a Vouchsafe library to read and write Identity Files.

---

## 2. Normative Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** have the meanings specified in BCP 14.

These meanings apply only when the words are in uppercase.

---

## 3. Terms

This specification uses these technical terms.

### 3.1 Identity File

An **Identity File** is a JSON object that contains a Vouchsafe URN and its key material.

### 3.2 Encrypted Private Key Blob

An **Encrypted Private Key Blob** is the binary data in the `encryptedPrivateKey` field.

The Identity File stores this binary data in Base64 format.

### 3.3 Cipher

A **cipher** is the authenticated encryption algorithm that encrypts the private key.

### 3.4 KDF

A **KDF** is a key derivation function.

The KDF derives an encryption key from a passphrase.

### 3.5 Binary String

A **binary string** is a length-prefixed sequence of bytes.

Section 7 specifies this encoding.

### 3.6 Base64

Unless this specification states otherwise, **Base64** means the standard Base64 alphabet defined in RFC 4648 §4.

Base64 data MUST include standard `=` padding when padding is required.

---

## 4. Identity File Structure

A Vouchsafe Identity File is a JSON object.

### 4.1 Unencrypted Identity File

An unencrypted Identity File has this form:

```json
{
    "urn": "urn:vouchsafe:bob.efxvsu2ehglkvh3qlewddsoufzwvofzvttfvrmal3a6x2jydeqcq",
    "keypair": {
        "publicKey": "MCowBQYDK2VwAyEAudRnAX9/lK9NKs4bVwyeRvu2m1cKZemo5bikEK2Fo90=",
        "privateKey": "BASE64_DER_PRIVATE_KEY"
    },
    "publicKeyHash": "efxvsu2ehglkvh3qlewddsoufzwvofzvttfvrmal3a6x2jydeqcq",
    "version": "1.4.0"
}
```

### 4.2 Encrypted Identity File

An encrypted Identity File has this form:

```json
{
    "urn": "urn:vouchsafe:bob.efxvsu2ehglkvh3qlewddsoufzwvofzvttfvrmal3a6x2jydeqcq",
    "keypair": {
        "publicKey": "MCowBQYDK2VwAyEAudRnAX9/lK9NKs4bVwyeRvu2m1cKZemo5bikEK2Fo90=",
        "encryptedPrivateKey": "BASE64_ENCRYPTED_PRIVATE_KEY_BLOB"
    },
    "publicKeyHash": "efxvsu2ehglkvh3qlewddsoufzwvofzvttfvrmal3a6x2jydeqcq",
    "version": "1.4.0"
}
```

### 4.3 Required Fields

An Identity File MUST contain these fields:

| Field | Type | Description |
|---|---|---|
| `urn` | string | The full Vouchsafe identity URN. |
| `keypair` | object | The key material for the identity. |
| `keypair.publicKey` | string | The Base64-encoded DER public key. |
| `publicKeyHash` | string | The lowercase, unpadded Base32 SHA-256 hash of the raw public key. |
| `version` | string | The Identity File format version. |

The `keypair` object MUST contain exactly one of these fields:

| Field | Type | Description |
|---|---|---|
| `privateKey` | string | The Base64-encoded DER private key. |
| `encryptedPrivateKey` | string | The Base64-encoded Encrypted Private Key Blob. |

`privateKey` and `encryptedPrivateKey` MUST NOT occur in the same Identity File.

---

## 5. Key Binding Check

The **Key Binding Check** verifies that all key and identity fields refer to the same Vouchsafe identity.

An implementation MUST perform this check before it accepts an Identity File.

The `publicKey` field MUST contain a DER-encoded Ed25519 SubjectPublicKeyInfo structure.

The Identity File MUST encode this DER data with standard Base64 as defined in RFC 4648 §4, with padding.

The `publicKeyHash` field MUST contain the hash of the raw Ed25519 public-key bytes.

To perform the Key Binding Check:

1. Decode `keypair.publicKey` from Base64.
2. Extract the raw Ed25519 public-key bytes from the DER SubjectPublicKeyInfo structure.
3. Calculate SHA-256 on the raw public-key bytes.
4. Encode the SHA-256 result with Base32.
5. Remove Base32 padding.
6. Convert the Base32 text to lowercase.
7. Compare the result with `publicKeyHash`.
8. Compare the hash part of `urn` with `publicKeyHash`.
9. If private-key material is available, get the public key that corresponds to the private key and compare it with `keypair.publicKey`.

All comparisons MUST be successful.

If a comparison is not successful, the implementation MUST reject the Identity File.

---

## 6. Unencrypted Private Keys

If `keypair.privateKey` is present, it MUST contain a DER-encoded PKCS#8 private key.

The Identity File MUST encode this DER data with standard Base64 as defined in RFC 4648 §4, with padding.

For Ed25519 private keys, implementations MUST parse the DER data as PKCS#8.

For example, a JavaScript implementation can load the decoded bytes with an API equivalent to:

```text
format = "der"
type   = "pkcs8"
```

After the implementation parses the private key, it MUST perform the Key Binding Check in Section 5.

If the Key Binding Check is not successful, the implementation MUST reject the Identity File.

---

# Vouchsafe Encrypted Private Key Format

## 7. Binary Encoding

The `encryptedPrivateKey` field contains standard Base64 text as defined in RFC 4648 §4, with padding.

When decoded, this text produces an Encrypted Private Key Blob.

### 7.1 Binary String Format

The binary format uses length-prefixed strings.

Each binary string has this format:

```text
uint32 length
byte[length] value
```

`length` is an unsigned 32-bit integer.

The integer MUST use network byte order. Thus, the most-significant byte occurs first.

The value of `length` is the number of bytes that immediately follow the length field.

A binary string can contain arbitrary binary data unless this specification gives a different requirement.

### 7.2 Algorithm Names

Algorithm names use printable ASCII characters.

Algorithm names are case-sensitive.

Implementations MUST compare algorithm names byte-for-byte.

An implementation MUST reject an algorithm name that it does not support.

---

## 8. Encrypted Private Key Blob

The Encrypted Private Key Blob has this structure:

```text
string ciphername
string kdfname
string kdfoptions
string encrypted
```

### 8.1 `ciphername`

`ciphername` identifies the cipher.

For example:

```text
aes256-gcm
```

The cipher specification defines the format of `encrypted`.

### 8.2 `kdfname`

`kdfname` identifies the KDF.

For example:

```text
pbkdf2-sha256
```

The KDF specification defines the format of `kdfoptions`.

### 8.3 `kdfoptions`

`kdfoptions` contains the parameters for the KDF.

The outer Encrypted Private Key Blob does not interpret these bytes.

The implementation MUST interpret these bytes according to `kdfname`.

### 8.4 `encrypted`

`encrypted` contains the cipher data.

The outer Encrypted Private Key Blob does not interpret these bytes.

The implementation MUST interpret these bytes according to `ciphername`.

---

## 9. Private-Key Plaintext

The plaintext for encryption MUST be the DER-encoded PKCS#8 private-key bytes.

Do not encrypt the Base64 representation of the DER data.

Thus, an unencrypted private key has this form:

```text
privateKey = Base64(DER)
```

An encrypted private key has this form:

```text
encryptedPrivateKey =
    Base64(
        EncryptedPrivateKeyBlob(
            Encrypt(DER)
        )
    )
```

The encrypted and unencrypted formats contain the same DER private-key data.

---

## 10. KDF Algorithms

This section defines KDF algorithms that Vouchsafe Identity Files can use.

### 10.1 `pbkdf2-sha256`

The algorithm name:

```text
pbkdf2-sha256
```

identifies PBKDF2 with HMAC-SHA-256.

The `kdfoptions` field has this structure:

```text
string salt
uint32 iterations
```

#### 10.1.1 Salt

`salt` MUST contain cryptographically random bytes.

Generate a new salt each time that you encrypt a private key.

For a new Identity File, the salt MUST contain at least 16 bytes.

#### 10.1.2 Iteration Count

`iterations` specifies the PBKDF2 iteration count.

Encode `iterations` as an unsigned 32-bit integer in network byte order.

When you decrypt a key, use the iteration count in `kdfoptions`.

Do not replace this value with the current default value of the implementation.

An implementation MAY change its default iteration count for newly encrypted keys.

A change to the default iteration count does not change the `pbkdf2-sha256` algorithm name.

#### 10.1.3 Passphrase Encoding

Encode the passphrase as UTF-8 before you give it to PBKDF2.

Do not apply Unicode normalization to the passphrase.

This rule is a compatibility requirement.

The same visible passphrase can have different Unicode byte sequences on different platforms or input methods.

In that case, decryption will fail.

Implementations MUST NOT add Unicode normalization to correct this condition because that change can break cross-implementation compatibility.

#### 10.1.4 Derived Key Length

When `pbkdf2-sha256` is used with `aes256-gcm`, derive 32 bytes of key data.

#### 10.1.5 `kdfoptions` Encoding Example

The following example shows the encoded structure for a 16-byte salt and 600,000 PBKDF2 iterations.

Assume the salt bytes are:

```text
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
```

The `kdfoptions` bytes are:

| Offset | Bytes | Meaning |
|---|---|---|
| `0` | `00 00 00 10` | Salt length: 16 bytes |
| `4` | `00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f` | Salt bytes |
| `20` | `00 09 27 c0` | Iteration count: 600,000 |

The total `kdfoptions` length in this example is 24 bytes.

The outer Encrypted Private Key Blob stores these 24 bytes as one length-prefixed `string`.

---

## 11. Cipher Algorithms

This section defines cipher algorithms that Vouchsafe Identity Files can use.

### 11.1 `aes256-gcm`

The algorithm name:

```text
aes256-gcm
```

identifies AES-256-GCM.

The `encrypted` field has this structure:

```text
string nonce
string ciphertext
string tag
```

#### 11.1.1 Nonce

For a newly encrypted key, generate a cryptographically random 12-byte nonce.

Do not use the same nonce again with the same encryption key.

#### 11.1.2 Ciphertext

`ciphertext` contains the encrypted DER-encoded PKCS#8 private-key bytes.

#### 11.1.3 Authentication Tag

`tag` contains the AES-GCM authentication tag.

For `aes256-gcm`, the tag MUST contain 16 bytes.

#### 11.1.4 `encrypted` Encoding Example

The following example shows the binary layout for:

* A 12-byte nonce.
* A 48-byte ciphertext.
* A 16-byte authentication tag.

The actual nonce, ciphertext, and tag bytes are example values only.

| Offset | Size | Meaning |
|---|---:|---|
| `0` | 4 bytes | Nonce length. Value: `00 00 00 0c`. |
| `4` | 12 bytes | Nonce bytes. |
| `16` | 4 bytes | Ciphertext length. Value: `00 00 00 30`. |
| `20` | 48 bytes | Ciphertext bytes. |
| `68` | 4 bytes | Tag length. Value: `00 00 00 10`. |
| `72` | 16 bytes | Authentication tag bytes. |

The total `encrypted` length in this example is 88 bytes.

The outer Encrypted Private Key Blob stores these 88 bytes as one length-prefixed `string`.

---

## 12. Associated Authenticated Data

AES-GCM MUST authenticate the algorithm information and the KDF options.

For `aes256-gcm`, construct the Associated Authenticated Data (AAD) from these encoded fields:

```text
AAD =
    encode_string(ciphername) ||
    encode_string(kdfname) ||
    encode_string(kdfoptions)
```

`encode_string(x)` means:

```text
uint32 length
byte[length] value
```

Use the exact encoded bytes from the Encrypted Private Key Blob.

Do not include the `encrypted` field in the AAD.

This rule authenticates:

* The cipher name.
* The KDF name.
* The KDF parameters.

If these values change, AES-GCM authentication MUST fail.

---

## 13. Encrypt a Private Key

Use this procedure when:

```text
ciphername = "aes256-gcm"
kdfname    = "pbkdf2-sha256"
```

1. Get the DER-encoded PKCS#8 private-key bytes.
2. Get the passphrase.
3. Encode the passphrase as UTF-8.
4. Do not normalize the Unicode text.
5. Generate a cryptographically random salt.
6. Select the PBKDF2 iteration count.
7. Encode `kdfoptions`.
8. Use PBKDF2-HMAC-SHA-256 to derive a 32-byte key.
9. Generate a cryptographically random 12-byte nonce.
10. Construct the AAD.
11. Encrypt the DER private-key bytes with AES-256-GCM.
12. Get the ciphertext and the 16-byte authentication tag.
13. Encode the `encrypted` field.
14. Encode the Encrypted Private Key Blob.
15. Encode the complete blob with standard Base64 as defined in RFC 4648 §4, with padding.
16. Put the Base64 text in `encryptedPrivateKey`.

The `kdfoptions` structure is:

```text
string salt
uint32 iterations
```

The `encrypted` structure is:

```text
string nonce
string ciphertext
string tag
```

The complete blob is:

```text
string ciphername
string kdfname
string kdfoptions
string encrypted
```

If the Identity File contains `encryptedPrivateKey`, do not include `privateKey`.

---

## 14. Decrypt a Private Key

Use this procedure to load an encrypted Identity File.

1. Decode `encryptedPrivateKey` from standard Base64 as defined in RFC 4648 §4.
2. Read `ciphername`.
3. Read `kdfname`.
4. Read `kdfoptions`.
5. Read `encrypted`.
6. Make sure that no bytes remain after the `encrypted` field.
7. Select the KDF from `kdfname`.
8. Decode `kdfoptions` according to the KDF specification.
9. Encode the passphrase as specified by the KDF.
10. Derive the encryption key.
11. Select the cipher from `ciphername`.
12. Decode `encrypted` according to the cipher specification.
13. Construct the AAD from the encoded algorithm fields.
14. Decrypt and authenticate the ciphertext.
15. Parse the result as a DER-encoded PKCS#8 private key.
16. Perform the Key Binding Check in Section 5.

If bytes remain after the last field, reject the Encrypted Private Key Blob.

If authenticated decryption is not successful, do not load the Identity File.

If the Key Binding Check is not successful, reject the Identity File.

---

## 15. Library Processing

Vouchsafe libraries SHOULD provide functions that read and write Identity Files.

Applications SHOULD use these library functions instead of directly processing the encrypted private-key format.

If `privateKey` is present, the library SHOULD load the Identity File without a passphrase.

If `encryptedPrivateKey` is present, the library MUST require the passphrase that is necessary to decrypt the private key.

The application does not need to know the cipher, the KDF, or their parameters.

---

## 16. Errors

Normal application code does not need to know the specific cause of a decryption failure.

An implementation SHOULD use the same decryption error for these conditions:

* The passphrase is incorrect.
* The ciphertext changed.
* The authentication tag is incorrect.
* The encrypted data is damaged.
* The decrypted data is not a valid private key.

An implementation MAY use different errors for these conditions:

* The JSON structure is not valid.
* The Encrypted Private Key Blob is not valid.
* The algorithm is not supported.
* A passphrase is necessary but is not supplied.

---

## 17. Future Expansion

The Encrypted Private Key Blob is designed to support future KDFs and ciphers without a change to the outer binary structure.

Future specifications MUST use established cryptographic algorithms.

Future specifications MUST NOT require novel cryptographic primitives for compatibility with this format.

The outer structure remains:

```text
string ciphername
string kdfname
string kdfoptions
string encrypted
```

### 17.1 Future KDF Support

A future KDF definition MUST specify:

* A unique `kdfname`.
* The binary structure of `kdfoptions`.
* The passphrase encoding.
* The method that derives key material.
* Applicable parameter limits.

For example, a future specification can define:

```text
kdfname = "scrypt"
```

with KDF options such as:

```text
string salt
uint32 N
uint32 r
uint32 p
```

This example does not define Vouchsafe support for `scrypt`.

A Vouchsafe specification MUST define the complete processing rules before implementations use a KDF name.

### 17.2 Future Cipher Support

A future cipher definition MUST specify:

* A unique `ciphername`.
* The binary structure of `encrypted`.
* The required encryption-key size.
* The authenticated encryption procedure.
* The AAD rules.

All ciphers defined for this format MUST provide authenticated encryption.

### 17.3 Supported Combinations

An implementation does not have to support every combination of defined KDFs and ciphers.

An implementation MUST reject an unsupported combination.

The algorithm names in the Encrypted Private Key Blob are format information.

They are not an algorithm-negotiation mechanism.

Applications SHOULD NOT show these values as cryptographic configuration options.

---

## 18. Version Requirements

The `version` field identifies the Vouchsafe Identity File format version.

Each Identity File version defines the KDFs and ciphers that conforming implementations MUST support.

An implementation MAY support additional algorithms that a later specification defines.

An implementation MUST NOT create an Identity File that uses an algorithm combination that is not defined for the file version that it writes.

### 18.1 Required Algorithm Support

| Identity File Version | Required KDF | Required Cipher | Required Combination |
|---|---|---|---|
| `1.4.x` | `pbkdf2-sha256` | `aes256-gcm` | `pbkdf2-sha256` + `aes256-gcm` |

A conforming implementation that supports encrypted Identity Files with version `1.4.x` MUST support the combination:

```text
pbkdf2-sha256 + aes256-gcm
```

A later version of this specification MAY add more required or optional algorithms.

Support for a future algorithm does not imply that version `1.4.x` Identity Files can use that algorithm.

---

## 19. Security Requirements

### 19.1 Keep Identity Files Private

An Identity File contains private cryptographic material.

This is true for an encrypted Identity File and for an unencrypted Identity File.

Do not use an Identity File as a public representation of a Vouchsafe identity.

Use the Vouchsafe URN when you must identify a Vouchsafe identity to another party.

### 19.2 Use Strong Passphrases

The security of an encrypted Identity File depends in part on the passphrase.

Applications SHOULD permit sufficiently strong passphrases.

Implementations SHOULD change their default KDF work factor when security requirements and available hardware change.

The KDF parameters in an existing Identity File MUST continue to control the decryption of that file.

### 19.3 Check KDF Parameters

An attacker can change data in an Identity File before an implementation starts decryption.

Some KDF parameters can cause large CPU or memory use.

Before you start a KDF, make sure that its parameters are in acceptable limits.

An implementation SHOULD reject parameter values that can cause excessive resource use.

### 19.4 Use Authenticated Encryption

All ciphers defined for this format MUST provide authenticated encryption.

Do not define a cipher that provides confidentiality without integrity protection.

### 19.5 Check the Private Key

Successful decryption does not prove that the private key belongs to the Identity File.

After decryption, perform the Key Binding Check in Section 5.

Reject the Identity File if the Key Binding Check is not successful.

### 19.6 Authenticate Encryption Parameters

For `aes256-gcm`, the AAD includes:

* `ciphername`
* `kdfname`
* `kdfoptions`

A change to these fields MUST cause authentication to fail.

---

## 20. Format Summary

A Vouchsafe Identity File has this JSON structure:

```json
{
    "urn": "...",
    "keypair": {
        "publicKey": "...",
        "encryptedPrivateKey": "..."
    },
    "publicKeyHash": "...",
    "version": "..."
}
```

`encryptedPrivateKey` is standard Base64 text as defined in RFC 4648 §4, with padding.

When decoded, it has this binary structure:

```text
string ciphername
string kdfname
string kdfoptions
string encrypted
```

`kdfname` defines the format of `kdfoptions`.

`ciphername` defines the format of `encrypted`.

The JSON does not expose the encryption parameters as configuration fields.

Vouchsafe libraries read and write the Identity File format.

---

## Appendix A. Initial Binary Structure

For Vouchsafe Identity File version `1.4.x`, the required encrypted-key combination has this logical structure:

```text
string "aes256-gcm"

string "pbkdf2-sha256"

string {
    string <random salt>
    uint32 <iterations>
}

string {
    string <12-byte nonce>
    string <ciphertext>
    string <16-byte authentication tag>
}
```

Encode the complete binary structure with standard Base64 as defined in RFC 4648 §4, with padding.

Put the result in `encryptedPrivateKey`.

---

## Appendix B. KDF Defaults

An implementation SHOULD select an appropriate PBKDF2 work factor when it creates a new encrypted Identity File.

Do not make the current default part of the binary format definition.

Store the selected iteration count in `kdfoptions`.

When you decrypt an existing Identity File, always use the stored iteration count.

This lets implementations increase the default work factor without making old Identity Files invalid.
