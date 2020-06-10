<pre>
    WIP: WIP-Gorka-Irazoqui-compact-multiproof
    Layer: Applications
    Title: Compact Merkle Multi-proof generation and verification
    Author: Gorka Irazoqui <gorka@witnet.foundation>
    Discussions-To: `#dev-general` channel on Witnet Comunnity's Discord server
    Status: Draft
    Type: Standards Track
    Created: 2020-06-11
    License: BSD-2-Clause
</pre>

## Abstract

This proposal introduces an efficient technique to present multiple proof of inclusions against the same merkle root. This turns to be crucial for the Witnet bridge node as it needs to proof that several identities belonged to a unique ARS merkle root stored in the Block Relay. This proposal improves upon [Compact-MultiProofs](https://arxiv.org/abs/2002.07648) by assuming non-binary merkle trees, as it is the case for Witnet.

## Rationale

It should be of no surprise that bridge nodes need to proof the membership of several ARS members against a single merkle root in order to validate that 2/3s of them signed a particular block/superblock. In the following sections we assume there exists an ARS merkle root against which we want to prove the belonging of several leaves.

The naive way to do this is to provide a proof for each of the members for which we want to prove the membership. This, however, quicly becomes expensive. Consider the following example:

<img src="./WIP-Gorka-Irazoqui-compact-multiproof/three_merkle_proofs.png<"></img>

In this example we want to verify the membership of T2, T3, and T5, which involves 9 hashes to be performed in total. Instead we can verify all of them providing the following values:

<img src="./WIP-Gorka-Irazoqui-compact-multiproof/merkle_multi_proof.png<"></img>

In this case we prove the mebership of leaves 2, 3 and 5 with the same proof. For this to happen, we need to provide H1, H4, H6 and H7,8. More than that, the membership of all members can be proved with 6 hashes, compared to the previous 9. This is what we call sparse merkle multiproofs. In this case, the assumptions is that the indices for those leaves we do not want to prove the membership for need to be given, alongside those that need to be prove. [Compact merkle proofs]((https://arxiv.org/abs/2002.07648)) instead only make use of the indices to be verified. 

## Specification
Compact merkle proofs have only been proposed in the context of binary trees. As the number of leaves in Witnet does not necessarily need to be a power of 2, the algorithm proposed in [Compact merkle proofs]((https://arxiv.org/abs/2002.07648)) needs to be changed. These are the changes we propose to such algorithm:

**Compact merkle proof generation**:

Every leaf node has an index from 0 to N, where N is the total number of leaves in a Merkle tree. We first determine the index for every element that takes part in the multiproof. In the case of our example in figure 7, that would be indices [2, 3, 8, 13]. Let’s name this array *A* and let’s name the "Merkle layer" on which we operate as *L*. In this case, *L* will change depending on the length of the tree. After determining these indices, we run the following steps recursively until termination (or until indices has one single member in the verification case):

- Store the depth of the merkle tree in *depth*
- Calculate the highest multiple of 2 of *depth*, and store it in *higest_two_mul*. Our layer L will be 2^*higest_two_mul*
- Compose an array *A'* with the number of indices from A smaller than *higest_two_mul*
- Compose an array *A''* with the number of indices from A higher or equal than *higest_two_mul*
- For each of the indices in *A'*, take the index of its immediate neighbor in a hypothetical layer L, and store the given element index and the neighboring index as a pair of indices (an "immediate neighbor" is the leaf index right next to a target leaf index that shares the same parent). Let’s name this array *B*.
- Remove any duplicate from *B* to form *B'*.
- Take the difference between the set of indices in *B'* and *A'* and append the hash values for the given indices.
- We take all the even numbers from *B'*, and divide them by two. We assign the newly computed numbers to *A*.
- We take all the numbers of *A''*, compute *index = index-(highest_two_mul//2)* and assign them to A.
- Substract *depth = depth - (highest_two_mul//2)*
- Repeat the above steps with the newly assigned variables A and L until you reach the root of the tree

<img src="./WIP-Gorka-Irazoqui-compact-multiproof/compact_proof_generation.png<"></img>

The proof at the end must contain the indices of the elements used for the multiproof, as well as the gathered hashes inside M.

**For a compact Merkle multiproof verification**:
We require the indices of the elements used for a multiproof (Let’s name this array A), the corresponding hashes for the elements used for a multiproof, as well as the hashes of the multiproof M. Array A in case of our example 8 would be [2, 3, 8, 13]. We first need to sort the *k* element hashes in increasing order according to *A*. Let’s name this sorted array *E*. To verify  generated multiproof, we run the following steps recursively until termination:

- Store the depth of the merkle tree in *depth*
- Calculate the highest power of 2 of +depth*, and store it in *higest_two_power*. Our layer L will be *2^higest_two_power*
- Compose an array *A'* with the number of indices from *A* smaller than *higest_two_power*
- Compose an array *A''* with the number of indices from *A* higher or equal than *higest_two_power*
- For each of the indices in *A'*, take the index of its immediate neighbor in the layer *L*, and store the given element index and the neighboring index as a pair of indices (an "immediate neighbor" is the leaf index right next to a target leaf index that shares the same parent). Let’s name this array *B*.
- After computing *B*, we check for duplicate index pairs inside it. If two pairs are identical, we hash the corresponding values (that have the same indices) inside *E* with one another. If an index pair has no duplicates, we hash the corresponding value inside *E* with the first hash inside *M*. If a value inside *M* was used, we remove it from M. All the newly generated hashes are assigned to a new *E* that will be used for the next iteration.
- Copy the values associated with indices *A''* from *E* to the new *E*.
- Repeat the above steps until you reach the root of the tree

<img src="./WIP-Gorka-Irazoqui-compact-multiproof/compact_proof_verification.png<"></img>




