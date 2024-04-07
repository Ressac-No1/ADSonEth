# Study report: Stateless Verkle Tree scheme and its potential optimization

## Approach to statelessness
Ethereum nodes are removing their storage burden. In the upcoming stateless Ethereum, nodes record neither contract codes, code execution transcripts, nor data permanently stored in contracts on the blockchain. Instead, transaction senders generate **SNARK proofs** of codes, transcripts, and storage data access and update event lists during transaction execution, and submit the SNARKs to be verified by the nodes.

In practice, a transaction in the stateless version of Ethereum is described as an *execution witness* that contains the following state access and modification events in the context of the transaction execution with a SNARK to prove its authenticity:
- codes of created contracts
- transcript of the executed codes
- transferred values and balance changes
- EVM logs
- nonce changes
- **all storage slot read and write events**

The default underlying data structure of Ethereum storage, Merkle-Patricia Tree (MPT), has an overhead of witness and proof size while convincing a storage slot access or update event that occurs in a transaction — all the node hashes of a root-to-leaf path in an MPT must be involved in the witness, and to prove the correctness, hashes of all sibling nodes along the path are required. If the (hashed) key of a leaf has length *L* and the trie alphabet has length *q* (thus the trie is *2^q*-ary), then the witness size is Θ(*L/q*) and the proof size is Θ(*2^qL/q*) per slot event. Whereas *L* is probably large, witnesses and proofs on MPT may not be succinct enough to support statelessness.

## Verkle tree: a novel structure
In the verkle tree, a trie-like data structure proposed by Kuszmaul, a visit from the root to any node can be verified by a much more succinct proof than MPT because the link between every pair of adjacent nodes along the path is converted into a **polynomial commitment** in the proof and checked by **opening** such a polynomial commitment. If the proof of each link has a constant size, the entire proof size of a storage slot access event is O(*L/q*).

The cryptographic protocol of the verkle tree scheme refers to this article of D. Feist (https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html). A brief summary:
- The subtree of branch node `A` is committed via a polynomial `f_A`, in which `f_A(i)` is equal to the hash of the committed value of the child of `A` with edge label `i` for each letter `i` of the alphabet;
- The KZG polynomial commitment is applied in the proof of edge `i` of `A`, which proves that `g_A(x) = (f_A(x)-f_A(i))/(x-i)` is a polynomial;
- `f_A` is committed via either the Pedersen or Bulletproof polynomial commitment scheme;
- The `g` polynomials of all edges in a root-leaf path can be linearly combined into one polynomial with a "multiproof" to open its commitment. This trick reduces the prover time cost from O(*2^qL/q*) to O(*2^q*);
- The verifier checks the proof via bilinear pairing in O(*L/q*) computation time.

## Storage slot update in the verkle tree scheme
In Vitalik’s blog, an article demonstrated an optimized way to recalculate the commited value of a branch node responding to a storage slot update event with time cost O(1) rather than brute-force recalculation of O(*2^q*).

As the committed polynomial `f_A` of node `A` is indeed determined by the *2^q* evaluations `f_A(0), f_A(1), ..., f_A(2^q-1)`, it can be directly described as the linear combination of the *2^q* Lagrange basis: `l_i(x) = ∏(x-j)/(i-j)` for all `j≠i`: `f_A(x)=∑ f_A(i)l_i(x)`. Since `f_A(i)` represents the committed value of `A`'s child `i`, in the maintenance of a storage slot update, only one item `f_A(i)l_i(x)` of the sum changes for each node `A` in a root-leaf path. Therefore, if the polynomial commit scheme used in the commitment of `f_A` is *additive homomorphic*, then the commitment of `f_A` can be updated in O(1) computations with the help of the value change of `f_A(i)` and the preprocessed committed value of `l_i(x)`.

The proof size of a storage slot update event is also reduced to O(*L/q*), as it contains only the root-leaf path and the updated committed value of the nodes along the path.

## Potential optimization: proof reuse
### Our supposition: there may be more optimized Ethereum data storage models in which the proof size and verification time of a storage slot access event are both sublinear in the tree depth!
Note that the bottleneck of a proof is **all the node polynomial commitments** along a root-leaf path. If there has been a visit to a leaf node (representing a storage slot) `A` or one of `A`'s siblings, and the committed value of nodes along the path from the root to `A` has not been updated since then, the proof can be reused for the second visit of `A`. Even if the committed value has been updated, the renewed commitments may be contained in the proof of that storage slot update event, which is also reusable.

In EIP-4762, the gas reformation proposal for the verkle tree upgrade also indicates a much lower gas cost (200 compared to normal `SLOAD` gas cost 2100) for the reaccess of adjacent (leaves as siblings in the tree) storage slots, which probably considers proof reuse optimization.

**The next stage of our research project will explore the potential of proof reuse in efficient and gas-saving ADS storage schemes.**

### Reference
#### Verkle Trees (Original paper)
Kuszmaul, J. (2019). *Verkle trees*. Verkle Trees, 1, 1.
#### Verkle trees (in Vitalik’s blog)
https://vitalik.eth.limo/general/2021/06/18/verkle.html
#### VerkleInfo: Verkle Trees for Statelessness
https://verkle.info
#### EIP-4762: Statelessness gas cost changes
https://eips.ethereum.org/EIPS/eip-4762
#### EIP-6800: Ethereum state using a unified verkle tree
https://eips.ethereum.org/EIPS/eip-6800

