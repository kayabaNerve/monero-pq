# Post-quantum Monero

An under-development post-quantum alternative to the Monero protocol.

## The Minimum Viable Protocol

### Hash Function

We presume a collision-resistant hash function, its API denoted `H(v...)`, with
a compact output (48 or 64 bytes).

### The Commitment Scheme

We presume a post-quantum, additively homomorphic commitment scheme where
committing to any randomness of sufficient entropy achieves perfect hiding (and
the scheme itself if computationally binding). We denote this API as `C(v...)`,
with the output being in the same domain as `v`.

Ajtai commitments are additively homomorphic, binding, and hiding when a random
vector is additionally hashed (implied for a CRH which is as CR as optimal for
its output length, as stated within the LatticeFold paper). If it's perfectly
hiding needs to be followed up on.

https://eprint.iacr.org/2016/997 presents an additively homomorphic,
statistically hiding (hiding even when the adversary is allowed non-polylog
algorithms) commitment with just 9 KB per commitment.

### The Signing Scheme

We presume a post-quantum signature scheme. We prepare for one of two scenarios:

A) The signature scheme does not have re-randomizable keys.
B) The signature scheme does have re-randomizable keys.

A signature scheme with threshold multisignatures is strongly preferred.

At this time, Raccoon has been reviewed and there exists a threshold
multisignature protocol with an identical verification function. We would have
to review its assumptions, and why it wasn't a top candidate within the NIST
competition, yet if it passes our review and there's a non-interactive
re-randomization protocol, it would be clearly optimal.

Without re-randomization, and without a threshold multisignature protocol,
proving knowledge of the opening of a commitment would be acceptable (and have
much clearer security). This collapses the signing scheme to the commitment
scheme.

Raccoon, with a non-interactive re-randomization, would be preferred.

### The Key-Exchange Mechanism

We presume a post-quantum key-exchange mechanism (KEM). We bound it to not be
subject to any patent claims.

NTRU (Prime) would likely be the most performant and reviewed candidate. Classic
McEliece has notably small ciphertexts and fast decryption. Since the
key-exchange key is never placed on-chain, it's large size is potentially
acceptable.

### Addresses

An address is derived as followed:

1) Sample the signature verification key `S` (the 'spend' key).
2) Sample the key-exchange key `V` (the view key).
3) Sample randomness for the signature verification key `r0, r1`.
4) The address is the tuple `(C(r0, r1, S), V)`.

### Structures

We define an input as having one component:

- `Nullifier`: A nullifier, preventing the prior output from being double spent.

We define an output as having one component:

- `Commitment`: A commitment to the key which is allowed to spend this output
  (`K`) and the amount this output is for (`A`).

Then we define a transaction kernel as a set of inputs and outputs.

### Derivation of the Output Commitment

1) Sample randomness `r`.
2) Utilize the KEM so the recipient can recover `r`.
3) Derive the key commitment randomness `k` from `r`, binding to this
   transaction's nullifiers.
4) `K` is `address.0 + C(k, 0, 0)`.
5) Derive the key commitment randomness `a` from `r`, binding to this
   transaction's nullifiers.
6) `A` is `C(a, amount)`.
7) The output commitment is `H(K, A)`.

### Proofs

The protocol now necessitates five relationships be maintained.

1) The output spent is an output present on the blockchain (`Membership`).
2) The output spent hasn't been prior spent (`Linkability`).
3) The output is authorized to be spent in this transaction (`SpendAuth`).
4) The sum of the inputs is the sum of the outputs (`Balance`).
5) No outputs cause an overflow of the commitment (`Range`).

#### Membership

We define a Merkle tree of all output commitments on the blockchain.

The proof would output a pair of values `K', A'`, which are re-randomizations
of `K, A`. The proof starts by de-randomizing `K', A'` to `K, A`, and then
performs `H(K, A)` for the leaf to prove is within the Merkle tree. Finally, the
actual Merkle tree proof is performed.

For `K'`, the re-randomization occurs _in the second position_. This preserves
the original randomness while still re-randomizing the commitment.

This allows performing membership with only knowledge of the output to be spent.

#### Linkability

Addresses only have blinded commitments to spend keys. The derivers of outputs
never learn the spend keys accordingly. This lets us define the nullifier as
`H(C(r, S))`, where `r` is the randomness _in the first position_ of the key
commitment.

This does not require the private spend key to determine the nullifer.

### SpendAuth

If the signing scheme supports re-randomization, two proofs are used:

1) Extract the signing key from the re-randomized output commitment from
`Membership`. Output its re-randomization.
2) Produce a traditional signature for the re-randomized key.

If the signing scheme does not support re-randomization, one proof is used to
prove a signature exists for the key embedded in the re-randomized output
commitment from `Membership`.

### Balance

`Balance` is done by proving the sum of all input commitments minus the sum of
all output commitments is a commitment to zero.

### Range

Each amount commitment is opened and proven to be within an acceptable range, as
to not cause overflows when summed.

## In Practice

The goal of this project is have an option for Monero to fallback onto whenever
needed. That means initially implementing the protocol with the simplest,
most-reviewed proofs which satisfy our needs, and from there improving.

The simplest option is presumably a ZK-STARK, using the original FRI, and a
Blake2s-384 circuit (using a non-additively-homomorphic commitment, as described
as possible in earlier versions of this document). For the signing scheme,
knowledge of a preimage would be used. For the key exchange, likely NTRU Prime.
All of this would work. Transactions would be on the scale of ~500 kB, there
would be no multisig, and anyone participating in any part of the proving gains
much more information than desirable, yet it'd work.

Such a protocol is ideally implemented, as a proof of concept, in 6-12 months.
