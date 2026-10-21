> [!NOTE]
> 🚧 **Work in Progress** - Feedback of any kind is welcome, stars speed up progress, and contributions are highly appreciated! Feel free to open an issue or submit a pull request.

# Salted Merkle Tree for Selective Disclosure

A Salted Merkle tree is a cryptographic commitment structure. Selective disclosure is achieved by revealing only the targeted leaf nodes data and their corresponding verification path while keeping undisclosed claims private. It enables a verifier to validate disclosed claims against a single Merkle root. The design prioritizes security, privacy, and computational efficiency.

The tree is binary and balanced, using unique padding hashes to handle missing siblings without explicitly constructing or storing all padding nodes. This reduces the space and computational overhead associated with materializing a fully padded tree. Each leaf hash is salted to prevent offline dictionary attacks against the underlying claims.

For `n` leaves, root computation takes `O(n)` time and `O(n)` space, while the tree depth is `O(log n)`. Verification path generation takes `O(n)` time and `O(n)` space. An individual verification path contains `O(log n)` hashes. For `m` disclosed claims, root reconstruction takes `O(m log m + p)` time, where `p` is the number of hashes in the verification path, and `O(m)` space, excluding the input claims, salts, indices, and path. When multiple leaves are disclosed, their verification paths can share common nodes, reducing the total proof size.

## Design Objectives

- Secure Verification: Uses distinct domain-separation prefixes for leaf data, internal nodes, and padding values to distinguish hash inputs by type. Verification recomputes the commitment according to deterministic hashing rules and compares the resulting root with the trusted commitment.
- Privacy: Keeps undisclosed contents private while allowing disclosed claims to be independently verified.
- Unlinkability: Preserves the privacy of the total number of claims and the distribution of undisclosed claims, preventing their disclosure through the selective disclosure process.
- Computational Efficiency: Supports bottom-up root computation and verification path generation with limited intermediate storage. For multiple disclosed elements, shared verification nodes can be reused to reduce proof size and verification overhead.
- Structural Simplicity: Uses deterministic, salted padding to handle incomplete node pairs according to a fixed construction rule, avoiding ad hoc tree restructuring.

## Cryptographic Properties

- Deterministic Construction: Defines consistent leaf hashing, internal-node hashing, and padding rules, ensuring reproducible commitments for the same ordered input.
- Efficient Selective Disclosure: Enables verification of disclosed elements using sibling hashes and positional information to reconstruct and verify the Merkle root. Multiple disclosed elements share common verification nodes. 
- Salted Padding: Uses precomputed, salted padding hashes to handle non-power-of-two leaf counts consistently while keeping the total number of claims private. To the verifier, padding nodes are indistinguishable from other nodes and leafs.

These properties provide a deterministic and efficient basis for privacy-preserving selective disclosure, enabling verification of disclosed data while keeping undisclosed leaf contents and the total number of claims private, subject to the security of the underlying hash function and the confidentiality of the disclosed data itself.

## Datasets

Any dataset that supports a deterministic, unidirectional mapping between its native representation and an ordered sequence of canonicalized claims can be used for selective disclosure using a salted Merkle tree.

A claim is an atomic, canonicalized unit of information derived from a dataset that can be independently represented, committed to, and selectively disclosed. Each claim is encoded in a deterministic canonical form so that equivalent claims produce identical representations.

> Note: This document specifies algorithms for datasets represented as JSON, JSON-LD, and RDF (N-Quads). See _Appendix_.

## Actors

Conceptual separation among the Issuer, Holder, and Verifier isolates trust domains. The Issuer establishes cryptographic commitment via digital signing, the Holder executes selective disclosure, and the Verifier performs stateless validation. The three-party model enforces cryptographic separation of duties and ensures that the Verifier verifies claims via non-interactive path validation without establishing communication channels with the Issuer, thereby eliminating online tracking and preserving unlinkability across protocol sessions.

> Note: The Issuer and Holder roles may coincide as a single entity in self-issued scenarios.

### Issuer

An entity that instantiates a cryptographic commitment scheme over a dataset by serializing claims into a canonical sequence, mapping them to a sparse vector of dimension M via a keyed pseudorandom slot-distribution function, computing a Salted Merkle tree root, and generating a digital signature over the root. The Issuer transmits the signed root, verification key, HMAC secret key `K`, leaves dimension `M`, and dataset to the Holder.

- Flattens the dataset into an ordered list of claims.
- Distributes the claims across an array of size `M` using the secret-keyed HMAC slot-distribution algorithm.
- Generates the Salted Merkle root.
- Signs the Merkle root using the issuer's private key.
- Shares the following with the holder:
  - The signed Merkle root.
  - The issuer's public key.
  - The secret key `K` and leaves array size `M`.
  - The dataset.

> Node: The HMAC secret key `K` is never shared with the verifier; it is used to hide the original claims distribution and to add entropy to the leaf hashes, preventing dictionary attacks and brute-force attacks.
> 
> The issuer uses its asymmetric key pair and signature scheme to sign the commitment. The issuer's private key must remain strictly uncompromised and unshared.

### Holder (disclosing party)

An entity acting as a cryptographic prover in a selective disclosure protocol that receives the dataset, HMAC secret key `K`, array dimension `M`, and signed Merkle root from the Issuer. The Holder is responsible for serializing the dataset into a canonical sequence, mapping claims to a sparse vector of dimension `M` via the keyed slot-distribution function, computing cryptographic verification path for a subset of targeted claims, and transmitting the proofs and public parameters to the Verifier.

- Flattens the dataset into an ordered list of claims.
- Distributes the claims across an array of size `M` using the same HMAC slot-distribution algorithm and secret key `K`.
- Selects the claims to disclose.
- Generates the Salted Merkle tree verification path for the selected claims.
- Shares the following with the verifier:
  - The selected claims in their dataset representation.
  - The selected claims salts and assigned slot indices.
  - The corresponding verification path.
  - The issuer's signed Merkle root.

### Verifier

An entity that acts as a verifier in a selective disclosure protocol, responsible for authenticating the subset of claims revealed by the Holder. The Verifier reconstructs the Salted Merkle tree root from the disclosed claims, their respective salts, slot indices, and verification path, and cryptographically validates the issuer's digital signature over the reconstructed root using the issuer's public key.

- Flattens the relevant dataset into an ordered list of claims.
- Hashes the disclosed claims and assigns them to their slots using the provided salts and slot indices.
- Reconstructs the Salted Merkle root from the disclosed claims and verification path.
- Verifies the issuer's signature on the reconstructed Merkle root using the issuer's public key.
- Accepts the disclosed claims only if signature verification.

### Example

The example shows a tree with 11 claims, padded to an even number, i.e., 12 leaves. The underlying claims for leaves 1, 2, 8, and 11 are disclosed, while the remaining claims stay private. The verification path include the hashes for leaves 7 and 12, nodes 3:4, 5:6, 9:10 and P2 (depth-based padding level 2), allowing the verifier to reconstruct and verify the root.

![Example #1](example-1.svg)

## Algorithms

The selective disclosure process involves three core algorithms, used by the issuer, the holder (the disclosing party), and the verifier:

- Root Computation: The issuer computes a Merkle root from the complete, ordered set of leaf hashes. The root serves as a commitment to the dataset.
- Verification Path Generation: The holder (the disclosing party) selects the leaves to reveal and generates their verification path. The path provide the additional hash values and positional information needed to verify the disclosed leaves without revealing the remaining leaves.
- Root Reconstruction and Verification: The verifier uses the disclosed leaves and their verification path to reconstruct the Merkle root. The disclosed data is accepted as consistent with the original commitment if the reconstructed root matches the trusted root.

The algorithms enable the issuer to commit to a dataset, the disclosing party to reveal selected leaves, and the verifier to check them against the original commitment.

### Root Generation

**Input**: An ordered sequence of `n` leaf hashes, where `n` is even and `n > 1`.

**Output**: The Merkle root.

1. Compute the tree depth as `depth = ceil(log2(n))`.
2. Set the current `level_size` to `n`.
3. For each `level` from `0` to `depth - 1`:
   1. Process the current level in adjacent pairs, from left to right, using `level_size`.
   2. For each pair, compute the parent hash as `H(0x01 || left || right)`.
   3. If the final node has no right sibling, use the depth-based padding hash for the current `level` as its right sibling and increment the current `level_size` to account for the padded node.
   4. Store each parent hash in the same array at index `floor(i / 2)`, overwriting the entries of the previous level from left to right, where `i` is the index of the left child.
   5. Update the current `level_size` to `ceil(level_size / 2)`.
3. Return the hash at index zero as the Merkle root.

The algorithm reuses a single array to store successive tree levels, avoiding the need to allocate a separate array for each level. It terminates when the current level contains exactly one hash.

#### Complexity

Let `n` be the number of leaves:

- Time Complexity: `O(n)`. The algorithm hashes each leaf once and computes one parent hash per node pair, including pairs completed using padding. The total number of internal-node hashes is `n - 1`.
- Space Complexity: `O(n)`. The algorithm stores leaf hashes in a single array and reuses that array for successive tree levels, avoiding additional arrays proportional to the number of leaves.

These bounds assume fixed-size hash outputs and constant-time hash operations.

### Verification Path Generation

**Input**: An ordered sequence of `n` leaf hashes, where `n` is even and `n > 1`, and a selected leaf hashes are not set. Only not selected leaf hashes are computed and present.

**Output**: An ordered sequence of verification path hashes.

1. Compute the tree depth as `depth = ceil(log2(n))`.
2. Set the current `level_size` to `n`.
3. For each `level` from `0` to `depth - 1`:
   1. Process the current level in adjacent pairs, from left to right, using `level_size`.
   2. If the final node has no right sibling, use the depth-based padding hash for the current `level` as its right sibling and increment the current `level_size` to account for the padded node.
   3. For each pair where `i` is the index of the left child:
      1. If the left child is not set and the right child is present, append the right child to the verification path and unset the parent entry at index `floor(i / 2)`.
      2. If the right child is not set and the left child is present, append the left child to the verification path and unset the parent entry at index `floor(i / 2)`.
      3. If the left child is present and the right child is present, compute the parent hash as `H(0x01 || left || right)` and store it at index `floor(i / 2)`.
      4. Otherwise, unset the parent entry at index `floor(i / 2)`.
   4. Update the current `level_size` to `ceil(level_size / 2)`.
4. Return the verification path hashes.

The algorithm reuses a single buffer to store successive tree levels, promoting only nodes that were not selected nor already added to the verification path, avoiding the need to allocate a separate array for each level and unnecesary computations.

#### Complexity

Let `n` be the number of leaves:

- Time Complexity: `O(n)`. The algorithm processes the nodes at each tree level, performing constant-time checks and updates for each node. Since the number of nodes decreases geometrically at each level, the total number of node operations across all levels is `O(n)`. Hash computations are performed only for undisclosed leaves and their parent nodes.
- Space Complexity: `O(n)`. The algorithm reuses a single array to store successive tree levels and stores the verification path, which can contain up to `O(log2(n))` hashes for a single selected leaf.

These bounds assume fixed-size hash outputs, constant-time hash operations.

### Root Reconstruction

**Input**: A sequence of disclosed claims, their corresponding salts and original leaf indices, and a verification path hashes in verification order.

**Output**: The reconstructed Merkle root.

1. Initialize `mapping` with the indices of the disclosed claims and sort it by ascending original leaf index.
2. Initialize `buffer` to hold the disclosed leaf hashes and set `level_size` to the number of disclosed claims.
3. For each disclosed claim, compute its leaf hash as `H(PREFIX_LEAF || salt || claim)` and store it in `buffer`. Update the corresponding `mapping` entry to the claim's original leaf index.
4. While `level_size > 1` or unconsumed verification path hashes remain:
   1. Set `next_level_size = 0`.
   2. Process the current level from left to right, using the mapping to determine each node's position.
   3. For each node at position `node_index`:
      1. If `node_index` is odd, consume the next verification path hash as the left sibling, use the current buffer entry as the right child, and compute the parent hash as `H(0x01 || left || right)`.
      2. Otherwise, if the right node is present and its mapping entry equals `node_index + 1`, compute the parent hash from the current node and the next node as `H(0x01 || left || right)`, and advance iterator past both children.
      3. Otherwise, consume the next verification path hash as the right sibling, use the current entry as the left child, and compute the parent hash as `H(0x01 || left || right)`.
      4. Store the parent hash at index `next_level_size` in the buffer and store `floor(node_index / 2)` at the corresponding mapping entry.
      5. Increment `next_level_size`.
   4. Set `level_size` to `next_level_size`.
5. Return `buffer[0]` as the reconstructed root.

The algorithm sorts disclosed claims by their original leaf indices and reconstructs successive tree levels in place using `buffer` and `mapping` arrays sized to the number of disclosed claims, consuming verification path hashes only when a sibling is not among the disclosed nodes being processed.

#### Complexity

Let `m` be the number of disclosed claims and `p` the number of verification path hashes:

- Time Complexity: `O(m * log2(m) + p)`, Sorting takes `O(m * log2(m))`. During reconstruction, the number of nodes processed decreases at each level as pairs are combined into parents, so the total number of nodes processed across all levels is `O(m)`. Each of the `p` verification path hashes is consumed at most once, adding `O(p)` time.
- Space Complexity: `O(m)`, The algorithm uses a mapping array and a buffer proportional to the number of disclosed claims, reusing both arrays to store successive tree levels.

These bounds assume fixed-size hash outputs and constant-time hash operations.

### Leaf Hashes Computation

**Input**: An ordered sequence of `n` claims, their salts, a mapping from claim positions to leaf positions, and optional sorted indices of claims to omit.

**Output**: An ordered sequence of leaf hashes, with selected claims left unassigned and a padding leaf added if `n` is odd.

1. Initialize an array of size `n`, adding one extra slot if `n` is odd.
2. For each claim at index `i`:
   1. If no claims are selected for omission, or `i` does not match the next selected index, compute `H(PREFIX_LEAF || salts[i] || claims[i])` and store it at the leaf position specified by the mapping.
   2. Otherwise, omit the claim's leaf hash and advance to the next selected index.
3. If `n` is odd, compute the padding leaf as `H(PREFIX_LEAF || salts[n])` and store it at the position specified by the mapping for `n`.
4. Return the array of leaf hashes.

#### Complexity

- Time Complexity: `O(n)`, as each claim is processed once and at most one padding leaf is computed.
- Space Complexity: `O(n)` for the output buffer, including the optional padding leaf.

### Salt Generation, Claims Distribution, and Hash Computation

Salt Generation. Generate salts for all claims, ensuring the total number of salts is even by adding one additional entry if the number of claims is odd. Each salt is derived as `HMAC(K, 0x10 || uint32_be(i))`, where `i` is the origin claim index.

> Note: A unique, cryptographically random salt prevents attackers from efficiently guessing the contents of undisclosed leaves by precomputing hashes of likely values and comparing them against published hashes. This protection relies on keeping the salt secret for undisclosed leaves. The issuer and disclosing party can share a secret seed and deterministically derive each leaf's salt from its index, avoiding the need to store or transmit individual salts. When a leaf is disclosed, its salt must be revealed alongside the data so verifiers can recompute its hash.

Claims Distribution. Initialize the mapping with the original claim indices and sort them by their corresponding salts in ascending lexicographic order. The resulting mapping assigns each original claim index to its leaf slot in the tree.

> Note: Distributing claims according to positions derived from HMAC of their original indices using a secret key obscures the relationship between original claim indices and their positions in the Merkle tree, making it harder to infer the original ordering of claims or identify undisclosed claims by their tree positions.

Leaf Hash Computation. Compute each leaf hash as `H(0x00 || salt || claim)`. For a padding leaf, omit the data component, yielding `H(0x00 || salt)`.

Internal Node Hash Computation. Compute each parent hash as `H(0x01 || left || right)`, preserving the order of the left and right child hashes.

Level Padding. For a missing sibling at tree `level > 0`, derive the padding hash as `H(0x02 || HMAC(K, 0x11 || uint32_be(level)))`. The level-specific padding allows incomplete tree levels to be handled without explicitly materializing all padding nodes.

> Note: Domain separation prefixes for salt generation, leaf hashing, internal node computation, and level padding ensure that inputs from different structural contexts remain strictly separated, preventing hash collisions, type-confusion attacks, or second-preimage vulnerabilities between different node types and derivation steps.

## Appendix

### JSON Dataset

### Example

### Implementation Note

The `ceil`, `floor`, and `log2` operations can be implemented using integer arithmetic and bitwise operations, avoiding floating-point calculations and reducing computational overhead.

| Operation                    | Integer/bitwise equivalent                |
| ---------------------------- | ----------------------------------------- |
| `ceil(log2(n))`, for `n > 1` | `bit_length(n - 1)`                       |
| `floor(n / 2)`               | `n >>> 1` in Java                         |
| `ceil(n / 2)`                | `(n + 1) >>> 1` for non-negative integers |
| `2 ^ n`                      | `1 << n`                                  |
| `2 * ceil(n / 2)`            | `n + (n & 1)`                             |
| `n * 2`                      | `n << 1`                                  | 

Here, `bit_length(x)` is the number of bits in the binary representation of `x`. e.g. in Java, it can be computed as `32 - Integer.numberOfLeadingZeros(x)` for positive `int` values.

### Implementations
