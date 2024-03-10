# Study Report: Merkle-Patricia Trie

## Background
Ethereum states are stored in the database that every node backups and maintains. The database is built on several **Merkle-Patricia Tries** (MPTs).
### Patricia Trie
Trie (named after "re*trie*val") is a set of string data structures that support the operations of storage, lookup, insertion, deletion, and modification of data elements of \<key: string, value: any\>. Specifically, it is described as a tree with a letter on each branch, and each node matches the key composed by the letters on the path from the root to itself. A traditional trie is a *q*-ary tree with depth *L* and total size *O(NL)* to store *N* data elements in which keys are strings of length at most *L* on a size-*q* alphabet. A lookup of a key of length *L* on a traditional trie requires at most *L* node visits to reach the node matching the key or a non-existing (NULL) node that implies a "not found", and then to load the value of that node, modify its value, delete the node, or insert a new element with a key currently not exists. For insertion or deletion, a path of multiple nodes may be inserted or deleted to ensure that every leaf matches an existing key in the database.
### Merkle Tree
A Merkle tree encodes every node on the basis of its stored value and hashes them for authentication . The hash of a branch node is based on all its children’s hashes plus its encoded value and some specific metadata, which retains the data consistency of the entire subtree by verifying the *root hash* of the subtree— any disturbance of the structure or node values in the subtree affects the root hash. Therefore, a public verifier can ensure the data integrity of a set of elements by obtaining the root hash of the Merkle tree that stores all the elements without awareness of the full structure. It also enables a succinct proof of node *A*’s value in a Merkle tree by submitting the hashes of all nodes on the root-to-*A* path and all siblings of every node on the path. The verifier then confirms that the root hash is well synchronized and re-calculates the hashes along  path *A* to the root to ensure their correctness.
### Combination of both
The Merkle tree can be regarded as a scheme that encodes node values and authenticates the structure and stored data by hashing. Such a scheme can be implemented on any tree with data stored on nodes, including an equivalent of Patricia trie that transmits the substring labels on its edges to its nodes. The fundamental of MPT originates from the equivalent of Patricia trie, on which encoding and hashing algorithms are applied.

## MPT data storage scheme applied to Ethereum
In the MPT data storage scheme applied in Ethereum, the alphabet size *q*=16, with each letter represents a *nibble* or half-byte (0-15, 0x0-0xF) of the key as a byte array. There are three types of nodes in the MPT:
1. Branch nodes, which store the hash and 16 indices to all children;
2. Extension nodes, which store the encoded substring label as value, the index to the next node to visit as the only child, and the hash;
3. Leaf nodes, which store the encoded substring label as value and the hash.

Because the substring label’s raw text comprises nibbles whose number can be either even or odd and all the data have to be aligned in bytes, a substring is encoded by adding one or two flag nibbles as a prefix indicating the odd or even number of its raw length and whether the referred node is an extension or a leaf node:
- **00**: an extension node with even raw length (counted by nibbles, hereafter)
- **1**: an extension node with odd raw length
- **20**: a leaf node with even raw length
- **3**: a leaf node with odd raw length

## Implementation of the MPT scheme on Ethereum
One of the released implementations of the aforementioned MPT scheme is the `trie-db` crate (https://crates.io/crates/trie-db) authored by Parity Technology, which was applied in the Parity Ethereum core (https://github.com/openethereum/parity-ethereum/tree/v2.7.2-stable/ethcore). This implementation contains both a reminiscence of the MPT and a `proof` module to simulate the generation and verification of value authenticity proofs on the MPT.

### Reference
#### Merkle Patricia Trie - Ethereum Docs
https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/
#### Ethereum Data Structures
Jezek, K. (2021). *Ethereum Data Structures*. arXiv preprint arXiv:2108.05513.

