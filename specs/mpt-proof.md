# Merkle Patricia Trie (MPT) Proof

MPT circuit checks that the modification of the trie state happened correctly.

Let's assume there are two proofs (as returned by `eth getProof`):

- A proof that there exists value `val1` at key `key1` for the trie with root `root1`.
- A proof that there exists value `val2` at key `key1` for the trie with root `root2`.

The circuit checks the transition from `val1` to `val2` at `key1` that led to the change
of trie root from `root1` to `root2` (the chaining of such proofs is yet to be added).

The proof returned by `eth getProof` looks like (but there are various special cases):

- State trie branch 0 that contains 16 nodes, one of them being the hash of the branch below (State trie branch 1)

- State trie branch 1 that contains 16 nodes, one of them being the hash of the branch below

- ...

- State trie branch n

- State trie leaf that constains the info about account: nonce, balance, storageRoot, codeHash

- Storage trie branch 0 that contains 16 nodes, one of them being the hash of the branch below (Storage trie branch 1)

- Storage trie branch 1 that contains 16 nodes, one of them being the hash of the branch below

- ...

- Storage trie branch n

- Storage trie leaf that constains (part of) key `key1` and value `val1`

Let's for demonstration purposes simplify the proof above to only have two storage trie branches
and no state trie part:

- Storage trie branch 0 that contains 16 nodes, one of them being the hash of the branch below (Storage trie branch 1)
- Storage trie branch 1 that contains 16 nodes, one of them being the hash of the branch below
- Storage trie leaf that constains (part of) key `key1` and value `val1`

We split the branch information into 16 rows (one row for each node). The proofs looks like:

- Branch 0 node 0
- Branch 0 node 1
- Branch 0 node 2 (hash of Branch 1)
- ...
- Branch 0 node 15
- Branch 1 node 0
- Branch 1 node 1 (hash of a leaf)
- ...
- Branch 1 node 15
- Leaf

When `key1` is hashed and converted into hexadecimal value, it is a hexadecimal string of
length 64. The first character specifies under which position of Branch 0 is the node
corresponding to `key1`.
The second character specifies under which position of Branch 1 is the node
corresponding to `key1`. The remaining characters are stored in a leaf.
Let's say the first character is 2 and the second character is 1.
In our case, this means the hash of a leaf is Branch 1 node 1 and hash of Branch 1 is
Branch 0 node 2.

If we make a change at `key1` from `val1` to `val2` and obtain a proof after this change,
the proof will be different from the first one at Leaf, Branch 1 node 1, and Branch 0 node 2,
other proof elements stay the same.

To check the transition from `root1` to `root2` caused at `key1`, MPT circuit checks that both
proofs are the same except at the nodes that correspond to `key1` path (hexadecimal characters).
In proof 1, the root of Branch 0 needs to be `root1`.
In proof 2, the root of Branch 0 needs to be `root2`.
Furthermore, it needs to be checked that the nodes differ at indexes that
correspond to `key1` path.

To implement the constraints above, the two proofs are put in parallel in MPT rows.
Each branch row contains information of branch node from proof 1 and as well as from proof 2:

- Branch 0 node 0 before change || Branch 0 node 0 after change
- Branch 0 node 1 before change || Branch 0 node 1 after change
- ...
- Branch 0 node 15 before change || Branch 0 node 15 after change
- Branch 1 node 0 before change || Branch 1 node 0 after change
- ...
- Branch 1 node 15 before change || Branch 1 node 15 after change
- Leaf (before change)
- Leaf (after change)

## Circuit Layout

The two parallel proofs are called S proof and C proof in the MPT circuit layout.

The first 34 columns are for the S proof.
The next 34 columns are for the C proof.
The remaining columns are selectors, for example, for specifying whether the
row is branch node or leaf node.

34 columns presents 2 + 32. 32 columns are used because this is the length
of hash output.
Note that, for example, each branch node is given in a hash format -
it occupies 32 positions.
The first 2 columns are RLP specific as we will see below.

In the codebase, the columns are named:

- `s_rlp1`
- `s_rlp2`
- `s_advices` (32 columns)
- `c_rlp1`
- `c_rlp2`
- `c_advices` (32 columns)

### Branch

Branch comprises 19 rows:

- 1 init row with some RLP specific data and some selectors
- 16 node rows
- 2 extension node rows

| Row title           | Row example                                                       |
| --------------------- | ------------------------------------------------------------------ |
| Branch init            | `0, 1, 0, 1, 249, 2, 17, 249, 2, 17, 13`                                     |
| Node 0              | `0, 160, 215, 178, 43, ..., 0, 160, 215, 178,...`                                      |
| Node 1              | `0, 160, 195, 189, 38, ..., 0, 160, 195, 19, 38,...`                                      |
| Node 2 (empty)              | `0, 0, 128, 0, 0, ..., 0, 0, 128, 0, 0,...`                                      |
| ...              | ...                                      |
| Node 15              | `0, 160, 22, 99, 129, ..., 0, 160, 22, 99, 129,...`                                      |
| Extension row S              | `228, 130, 0, 149, 0,..., 0, 160, 57, 70,...`                                      |
| Extension row C              | `0, 0, 5, 0, ...,0, 160, 57, 70,... 0`                                      |

![branch](./img/branch.png)

The picture presents a branch which has 14 empty nodes and 2 non-empty nodes.
The last two rows are all zeros because this is a regular branch,
not an extension node.

Non-empty nodes are at positions 3 and 11. Position 11 corresponds to the key (that
means the key nibble that determines the position of a node in this branch is 11).

`s_advices/c_advices` present the hash of a branch child/node.

One can observe that in position 3: `s_advices = c_advices`,
while in position 11: `s_advices != c_advices`.

That is because this is due to the modification at `key1` from `val1` to `val2`,
the nibble 11 corresponds to `key1`.

We need `s_advices/c_advices` for two things:

- to compute the overall branch RLC (to be able to check the hash of a branch)
- to check whether `s_advices/c_advices` (at `is_modified` position)
  present the hash of the next element in a proof

Hash lookup looks like (for example for branch S):

```
lookup(S branch RLC, S branch length, s_advices RLC at the is_modified position in parent branch)
```

TODO: instead of 32 columns for `*_advices`, use only the RLC of `*_advices`.
To integrate `*_advices` RLC into the computation of the whole branch RLC, we
just need to compute `mult * *_advices_RLC` and add this to the current RLC value.

The layout then be simply:
`s_rlp1, s_rlp2, s_node_rlc, c_rlp1, c_rlp2, c_node_rlc`

#### Key RLC in branch

There are further columns (not shown in picture or table):

- `node_index`
- `is_modified`
- `modified_node`

In the example branch from the picture, `modified_node` would be 11.
`node_index` is simply an index running from 0 to 15. `modified_node` is 1 at
position 11, otherwise 0.

Let's see a branch layout where we replace 32 columns of `s_advices` with
one column `s_node_rlc` and 32 columns of `c_advices` with
one column `c_node_rlc` (this simplification will most likely be implemented):

```
s_rlp1_0, s_rlp2_0, s_node0_rlc, c_rlp1_0, c_rlp2_0, c_node0_rlc
s_rlp1_1, s_rlp2_1, s_node1_rlc, c_rlp1_1, c_rlp2_1, c_node1_rlc
...
s_rlp1_15, s_rlp2_15, s_node15_rlc, c_rlp1_15, c_rlp2_15, c_node15_rlc
```

With the three `modified_node` related columns:

```
s_rlp1_0, s_rlp2_0, s_node0_rlc, c_rlp1_0, c_rlp2_0, c_node0_rlc, 0 (node_index), 0 (is_modified), 11 (modified_node)
s_rlp1_1, s_rlp2_1, s_node1_rlc, c_rlp1_1, c_rlp2_1, c_node1_rlc, 1 (node_index), 0 (is_modified), 11 (modified_node)
...
s_rlp1_11, s_rlp2_11, s_node11_rlc, c_rlp1_11, c_rlp2_11, c_node11_rlc, 11 (node_index), 1 (is_modified), 11 (modified_node)
...
s_rlp1_15, s_rlp2_15, s_node15_rlc, c_rlp1_15, c_rlp2_15, c_node15_rlc, 15 (node_index), 0 (is_modified), 11 (modified_node)
```

While navigating through branches, key RLC is being computed in the last branch child
row. Let us say there are six branches with the following `modified_node` values:
3, 11, 15, 14, 2, 1.

These values present the first six nibbles of the key (after it is hashed).
That means the first three bytes of the key are: `3 * 16 + 11`, `15 * 16 + 14`,
`2 * 16 + 1`.

To compute (partial) key RLC, we need to compute:

```
(3 * 16 + 11) + (15 * 16 + 14) * r + (2 * 16 + 1) * r^2
```

Once we reach the leaf node, we take the rest of the nibbles there and compute
the full key RLC. This value needs to correspond to `key1` RLC.

In the last branch child row, we compute `11 * 16 * key_rlc_mult`
or `11 * key_rlc_mult` and add this value to the value in `key_rlc` row.

Columns (not shown in picture or table):

- `key_rlc_mult`
- `key_rlc`
- `sel1`
- `sel2`

Whether the factor 16 is used or not is determined by `sel1/sel2` columns.
`sel1/sel2` are boolean values and it holds `sel1 + sel2 = 0`.
The constraints for `sel1/sel2` are implemented in `BranchKeyChip`.

#### Branch init row

The first two columns specify whether S branch has two or three RLP meta bytes
(these bytes specify the length of the stream):

- `1, 0` means two RLP meta bytes
- `0, 1` means three RLP meta bytes

For example, a branch with two RLP meta bytes starts like:
`248, 81, 128, 128,... `
This means there are 81 bytes from position two onward in the branch RLP stream.

To check whether the length of the stream correspond to the length specified
with the RLP meta bytes, we use column 0. In each row we subtract the number
of bytes in a row. In the last row we checked whether the value is 0.

Note that branch node row can either have 33 bytes or 1 byte. 1 byte occurs
when the node is empty, in this case only the value 128 appears, which is stored
in column 2.

For example, a branch with three RLP meta bytes starts like:
`249, 1, 81, 128, 16, ... `
This means there are 1 * 256 + 81 bytes from position three onward.

Summary:

- cols 0 and 1: whether branch S has 2 or 3 RLP meta data bytes
- cols 2 and 3: whether branch C has 2 or 3 RLP meta data bytes
- cols 4 and 5: the actual branch S RLP meta data bytes
- col 6: the actual branch S RLP meta data byte (if there are 3 RLP meta data bytes in branch S)
- cols 7 and 8: branch C RLP meta data bytes
- col 9: the actual branch C RLP meta data byte (if there are 3 RLP meta data bytes in branch C)

TODO: selectors

#### Branch node rows

Each branch node row starts with 34 S proof columns and 34 C proof columns.

Example 34 S columns:

`0, 160, 215, 178, 43, ..., 23`

The first columns of S and C proof (`s_rlp1` and `c_rlp1`) in branch node rows are
used to check the RLP stream length.

The second columns (`s_rlp2` and `c_rlp2`) are for RLP encoding of the length
of the substream.
For non-empty rows, it is always 160, because this denotes the length of the
substream which is 32 (= 160 - 128). The substream in this case is hash of a
node.

When there is an empty node, the column looks like:
`0, 0, 128, 0, ..., 0`.

Empty node in a RLP stream is denoted only by one byte - value 128.
MPT circuit uses padding with 0s to simplify the comparison
between S and C branch. This way, the branch nodes are aligned horizontally
for both proofs.

For example, when a value is stored at a key that
hasn't been used yet, we will get an empty node in S branch and non-empty node
in C branch. We need to compare whether this change corresponds to the
key (key determines the index of the node in branch where change occurs).

#### Constraints

##### Constraint: hash of the branch is in the parent branch

`is_modified` selector denotes the position in branch which corresponds to the key
(the branch child where the change occurs).

The whole branch needs to be hashed and the result needs to be checked
to be in the parent branch. This is checked in `BranchHashInParentChip`
using a lookup which takes as an input:

- Random Linear Combination (RLC) of the branch
- parent branch `s_advices/c_advices` RLC at `is_modified` position

Hash lookup looks like (for S):
`lookup(S branch RLC, S branch length, s_advices RLC)`.

To simplify the constraints, the modified node RLC is stored in each
branch node. This is to enable rotations back to access the RLC of the modified node.
Thus, for example, when checking the branch hash to be in a parent branch,
we can rotate back to the last row in the parent branch and use the value from this
row for the lookup.

Let's see an example.
Let's say we have a branch where `modified_node = 1`. For clarity, let's
denote `s_rlp1, s_rlp2, c_rlp1, c_rlp2` simply with `_`.

```
_, _, b0_s_node0_rlc, _, _, b0_c_node0_rlc
_, _, b0_s_node1_rlc, _, _, b0_c_node1_rlc
...
_, _, b0_s_node15_rlc, _, _, b0_c_node15_rlc
```

Let's say the next element in a proof is another branch:

```
_, _, b1_s_node0_rlc, _, _, b1_c_node0_rlc
_, _, b1_s_node1_rlc, _, _, b1_c_node1_rlc
...
_, _, b1_s_node15_rlc, _, _, b1_c_node15_rlc
```

The hash of this second branch is in the parent branch at position 1.
Let `b1_s` be the RLC of S part of this second branch and
`b1_c` be the RLC of C part of this second branch.
Then:

```
hash(b1_s) = b0_s_node1_rlc
hash(b1_c) = b0_c_node1_rlc
```

Hash lookup like is needed (for S):
`lookup(b1_s, len(b1_s), b0_s_node1_rlc)`

We need a rotation to access `b0_s_node1_rlc`, but we cannot fix the rotation
as the `modified_node` can be any value between 0 and 15 - any of the following
values can appear to be needed: ` b0_s_node0_rlc, b0_s_node1_rlc, ..., b0_s_node15_rlc`.

For this reason there are two additional columns in all 16 branch children rows
that specify the `modified_node` RLC: `s_mod_node_hash_rlc` and `c_mod_node_hash_rlc`.

```
_, _, b0_s_node0_rlc, _, _, b0_c_node0_rlc, b0_s_mod_node_hash_rlc, b0_c_mod_node_hash_rlc
_, _, b0_s_node1_rlc, _, _, b0_c_node1_rlc, b0_s_mod_node_hash_rlc, b0_c_mod_node_hash_rlc
...
_, _, b0_s_node15_rlc, _, _, b0_c_node15_rlc, b0_s_mod_node_hash_rlc, b0_c_mod_node_hash_rlc
```

Now, we can rotate back to any of the branch noderen rows of `b0` to
access the RLC of the modified node.

##### Constraints: node hash in branch rows

We need to make sure `s_mod_node_hash_rlc` and `c_mod_node_hash_rlc`
is the same in all branch children rows.

```
s_mod_node_hash_rlc_cur = s_mod_node_hash_rlc_prev
c_mod_node_hash_rlc_cur = c_mod_node_hash_rlc_prev
```

##### Constraint: RLC of branch child at modified_node

At `modified_node` position, `s_mod_node_hash_rlc` and `c_mod_node_hash_rlc`
need to be the RLC of `s_advices` and `c_advices` of the node that corresponds
to the key (modified node).

```
rlc(s_advices) = s_mod_node_hash_rlc at position modified_node
rlc(c_advices) = c_mod_node_hash_rlc at position modified_node
```

##### Constraint: no change except at is_modified position

In all branch rows, except at `modified_node` position, it needs to hold:

```
s_node_rlc = c_node_rlc
```

With current implementation:

```
s_advices = c_advices
```

##### Constraint: is_modifed = 1 at modified_node, otherwise is_modified = 0

At `modified_node` position, it needs to hold:

```
is_modified = 1
```

At other positions:

```
is_modified = 0
```

##### Constraint: node_index increases by 1

```
node_index_cur = node_index_prev + 1
```

### Extension node rows

Extension node can be viewed as a special branch. It contains a regular branch
with the addition of a key extension. Key extension is set of nibbles (most
often only one or two nibbles) that "extend" the path to the branch.

The extension node element in proof (returned by `eth getProof`) contains
the information about nibbles and the hash of the underlying branch.

For example, the proof element of an extension node looks like:

`228,130,0,149,160,114,253,150,133,18,192,156,19,241,162,51,210,24,1,151,16,48,7,177,42,60,49,34,230,254,242,79,132,165,90,75,249`

130 means there are 2 (130 - 128) bytes compressing the nibbles.
These two bytes are
`0, 149`.
The two nibbles compressed are 9 and 5 (149 = 9 * 16 + 5).

The bytes after 160 present a hash of the underlying branch.

MPT layout uses `s_rlp1`, `s_rlp2`, and `s_advices` for RLP meta bytes and nibbles,
while `c_advices` are used for branch hash (`c_rlp2` stores 160 - denoting the number
of hash bytes).

There are two extension node rows - one for S proof, one for C proof.
However, the extension key (nibbles) is the same for both proofs, we don't need
to double this information.
For this reason, in C row, we don't put key into `s_rlp1`, `s_rlp2`, and `s_advices`,
we just put hash of C extension node underlying branch in `c_advices`.

However, in C row, we store additional witness for nibbles (because nibbles are
given compressed in bytes) into `s_rlp1`, `s_rlp2`, and `s_advices`.

For example:
`0, 0, 5, 0, 0, ...`

Here, 5 presents the second nibbles of 149 (see above).
Having the second nibble simplifies the computation of the first nibble.

Thus, the two extension rows look like:

```
228,130,0,149, 0, ..., 0, 160, S underlying branch hash
0, 0, 5, 0, ..., 0, 160, C underlying branch hash
```

There is bit of a difference in RLP stream when only one nibble appears.
In this case there is no byte specifying the length of the key extension
(130 in the above case).
For example, in the case below, the nibble is 0 (16 - 16):

`226,16,160,172,105,12...`

In this case special witnesses for nibbles are not needed.

#### Constraints

##### Constraint: selectors

There are six possible scenarios:

- extension node key contains only one nibble and `modified_node` needs to be
  multiplied by `16` for `key RLC`
- extension node key contains only one nibble and `modified_node` needs to be
  multiplied by `1` for `key RLC`
- extension node key contains even number of nibbles and `modified_node` needs to be
  multiplied by `16` for `key RLC`
- extension node key contains even number of nibbles and `modified_node` needs to be
  multiplied by `1` for `key RLC`
- extension node key contains odd number of nibbles (and more than 1) and `modified_node` needs to be multiplied by `16` for `key RLC`
- extension node key contains odd number of nibbles (and more than 1) and `modified_node` needs to be multiplied by `1` for `key RLC`

Extension node RLP encoding needs to be differently handled in different scenarios.
For example, in the the case of only one nibble, there is only one RLP meta byte
(key starts already in `s_rlp2`).

Key RLC information is packed together with information about number of nibbles to
reduce the expression degree.

It needs to be ensured that the selectors are boolean and their sum is `0` or `1`.
If it is `0`, there is a regular branch. If it is `1`, there is an extension node.
See `extension_node.rs` for the constraints.

Further, there are constraints in `extension_node_key.rs` that ensure the selector
corresponds to the actual

##### Constraint: extension node RLC is properly computed

Extension node RLC needs to be prperly computed for both, S and C.
This is done by taking into account each byte of the extension node.
The RLC is computed in two steps: the first
step computes bytes in `s_rlp1', 's_rlp2`, `s_advices` (stored in `acc_s` column),
the second step in `c_rlp1', 'c_rlp2`, `c_advices` (stored in `acc_c` column).

First step:

```
rlc_s = s_rlp1 + s_rlp2 * r + s_advices[0] * r^2 + s_advices[1] * r^3 + ... + s_advices[31] * r^33 
```

Constraint:

```
rlc_s = acc_s
```

Second step:

```
rlc = rlc_first + c_rlp1 * r_1 + c_rlp2 * r_1^2 + c_advices[0] * r_1^3 + c_advices[1] * r_1^4 + ... + c_advices[31] * r_1^34 
```

Constraint:

```
rlc = acc_c
```

Note that not all `s_advices` are always used. In the above example, there is only
`0, 149`. The rest of `s_advices` are 0s. To ensure `s_advices` are 0 for `i > 1`,
`key_len_lookup` function is used (see below for a more detailed description).

For S:
`lookup(S branch RLC (retrived from the last branch children row), S branch RLC length, c_advices RLC in extension row)`.

For C extension node, the RLC from the first step from S extension node row is reused.
The second step is analogous to the S row, but uses the values from C row.

##### Constraint: hash of the extension node is in the parent branch

It needs to be checked that the extension node RLC is `mod_node_hash_rlc`
in the parent branch.

```
lookup(acc_c, extension node S length, s_mod_node_hash_rlc::(rot))
lookup(acc_c, extension node C length, c_mod_node_hash_rlc::(rot-1))
```

##### Constraint: hash of the underlying branch is in the extension node c_advices

For S:
`lookup(S branch RLC (retrived from the last branch children row), S branch length, c_advices RLC in extension row)`.

### Storage leaf

There are 5 rows for a storage leaf.
2 rows for S proof, 2 rows for C proof
(it might be optimized to have only one row for a storage leaf).
1 row for cases when a leaf is turned into a branch or extension node.

`228, 159, 55, 204, 40,...`

______________________________________________________________________

`227, 161, 32, 187, 41, ..., 11`
`225, 159, 57, 202, 166, ..., 17`

## 0s in s_advices after substream ends

In various cases, `s_advices` are used only to certain point. Consider the example below:

```
228, 130, 0, 149, 0, ..., 0
```

In this example:

```
s_rlp1 = 228
s_rlp2 = 130
s_advices[0] = 0
s_advices[1] = 149
s_advices[2] = 0
...
s_advices[31] = 0
```

To prevent attacks on RLC, it needs to be checked that `s_advices[i] = 0` for `i > 1`:

```
s_advices[2] = 0
...
s_advices[31] = 0
```

The length of the substream is given by `s_rlp2`, it is `2 = 130 - 128` in the above example,
let us denote it by `len = 2`.

`s_advices[i]` are checked to be bytes.

Note that `(len - 1 - i) * s_advices[0] < 33 * 255` ensures `s_advices[i] = 0` for `i > len - 1`.

```
(len - 1) * s_advices[0] < 33 * 255
(len - 2) * s_advices[1] < 33 * 255
From now on, key_len < 0:
(len - 3) * s_advices[2] < 33 * 255 (Note that this will be true only if s_advices[2] = 0)
(len - 4) * s_advices[3] < 33 * 255 (Note that this will be true only if s_advices[3] = 0)
(len - 5) * s_advices[4] < 33 * 255 (Note that this will be true only if s_advices[4] = 0)
```

That is because when `len - i` goes below 0, it becomes a huge number close to field modulus.
Furthermore, `len` is at most 33.
If `len - i` is multiplied by `s_advices[i]` which is at most `255`, it will still be
bigger then `-32 * 255` which is much bigger than `33 * 255`.

See `key_len_lookup` in `helpers.rs` for the implementation.

## RLC multiplication factor after s_advices

As we have seen above,
in various cases, `s_advices` are used only to certain point. Consider the example below:

```
228, 130, 0, 149, 0, ..., 0
```

RLC is computed in two steps in such cases.
The first step computes bytes in `s_rlp1', 's_rlp2`, `s_advices` (stored in `acc_s` column),
the second step computes bytes in `c_rlp1', 'c_rlp2`, `c_advices` (stored in `acc_c` column).

First step:

```
rlc_first_step = s_rlp1 + s_rlp2 * r + s_advices[0] * r^2 + s_advices[1] * r^3 + ... + s_advices[31] * r^33 
```

Constraint

```
rlc_first_step = acc_s
```

Note that the RLC is computed and assigned in the `sythesize` function, the chips then
verify whether that the computation is correct.

In the next step:

```
rlc = rlc_first + c_rlp1 * r_1 + c_rlp2 * r_1^2 + c_advices[0] * r_1^3 + c_advices[1] * r_1^4 + ... + c_advices[31] * r_1^34 
```

Constraint:

```
rlc = acc_c
```

It also needs to be checked that `r_1` corresponds to `len`:

```
r_1 = r^(len+2)
```

This is checked using a lookup into a table:

```
(RMult, 0, 1)
(RMult, 1, r)
(RMult, 2, r^2)
(RMult, 3, r^3)
...
(RMult, 65, r^65)
```

The lookup looks like:

```
lookup(RMult, len+2, r_1)
```

See `mult_diff_lookup` in `helpers.rs` for the implementation.
