# STAC API - Merkle Tree Verification Extension

- **Title:** Merkle Tree Verification
- **Conformance Classes:**
  - `https://api.stacspec.org/v1.0.0/core` (required)
  - `https://api.stacspec.org/v1.0.0-beta.1/merkle-verification` (required)
- **Scope:** STAC API - Item Search & Collections
- **Extension Maturity Classification:** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  - [RFC 8785: JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785)
- **Owner**: @jonhealy1

## Introduction

This extension adds a **Data Integrity Layer** to the STAC API. It enables users to cryptographically verify that geospatial assets have not been tampered with, corrupted (bit rot), or swapped since they were originally published.

It defines the standard for advertising **Merkle Roots** (at the Collection level) and serving **Inclusion Proofs** (at the Item level).

### Conformance Advertisement
Implementations MUST advertise support by including the URI in the Landing Page (`/`) `conformsTo` array:

```json
{
  "conformsTo": [
    "https://api.stacspec.org/v1.0.0/core",
    "https://api.stacspec.org/v1.0.0-beta.1/merkle-verification"
  ]
}
```

## Technical Implementation

To ensure interoperability between different hashing implementations (e.g., Go, Python, Rust), strict adherence to canonicalization and hashing standards is required.

### 1. Canonicalization (JCS)
Before hashing any STAC Item, it MUST be transformed into its **Canonical Form** according to **[RFC 8785 (JSON Canonicalization Scheme)](https://www.rfc-editor.org/rfc/rfc8785)**.
* This ensures keys are sorted and whitespace is removed.
* **Exclusion:** When calculating the hash of a STAC Item, the `merkle:object_hash` field itself MUST be excluded from the input.

### 2. Hashing Algorithm
* **Standard:** SHA-256 (defined in FIPS 180-4).
* **Encoding:** All hashes MUST be represented as lowercase Hexadecimal strings.
* **Double Hashing:** To prevent length-extension attacks, the construction `Hash(Leaf)` implies `SHA256(SHA256(Canonical_JSON_Bytes))`.

## Data Model Extensions

### 1. Collection Object
The Collection acts as the Anchor.

| Field | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `merkle:root` | `string` | **Required** | The Hex-encoded Merkle Root Hash of the current state of the collection. |
| `merkle:root_method` | `string` | **Optional** | The hashing algorithm used. Allowed values: `sha256`. Defaults to `sha256` if omitted. |
| `merkle:tree_height` | `integer` | **Recommended** | The height of the tree (number of levels). Useful for client-side pre-allocation. |

### 2. Item Object
The Item acts as the Leaf.

| Field | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `merkle:object_hash` | `string` | **Required** | The SHA-256 hash of this Item's canonical JCS representation. |

### 3. Link Relations
| Relation | Description |
| :--- | :--- |
| `merkle-proof` | **Required.** Points to a JSON file containing the Merkle Inclusion Proof (siblings path). |

## Merkle Proof Structure

The resource pointed to by `rel="merkle-proof"` MUST adhere to the following JSON schema. This allows clients to verify the item without downloading the entire dataset.

**Proof JSON Example:**
```json
{
  "type": "MerkleProof",
  "version": "1.0.0",
  "leaf_hash": "8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4",
  "root_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "index": 4,
  "total_leaves": 15,
  "siblings": [
    {
      "hash": "a1b2c3d4...",
      "direction": "right"
    },
    {
      "hash": "c5d6e7f8...",
      "direction": "left"
    },
    {
      "hash": "99887766...",
      "direction": "left"
    }
  ]
}
```

**Verification Logic:**
1.  Start with `current_hash = leaf_hash`.
2.  Iterate through `siblings`.
3.  If `direction` is "right": `current_hash = Hash(current_hash + sibling.hash)`.
4.  If `direction` is "left": `current_hash = Hash(sibling.hash + current_hash)`.
5.  Assert `current_hash == root_hash`.

## Lifecycle & Operational Guidance

### Updating Roots (Dynamic Collections)
Most STAC Collections are dynamic (new satellite imagery is added daily). The Merkle Root is a commitment to a **specific state** of the collection.

* **Append-Only Strategy:** When new items are added, the Collection is considered to be in a new state. The `merkle:root` field in the Collection metadata MUST be updated to reflect the new global root.
* **Proof Validity:** Adding items to a Merkle Tree changes the path for *existing* items. Therefore, **adding new items requires regenerating proofs for existing items.**
    * *Note:* Implementers may choose to batch updates (e.g., "Daily Roots") to minimize regeneration costs.

## Endpoints

### `GET /collections/{id}/items/{id}/verify`
This endpoint performs server-side verification.

**Response:**
```json
{
  "verified": true,
  "timestamp": "2026-01-28T12:00:00Z",
  "algorithm": "sha256",
  "match_details": {
    "item_hash_match": true,
    "root_hash_match": true
  }
}
