---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Selective Disclosure CBOR Web Tokens (SD-CWT)"
abbrev: "SMD-CWT"
category: std

docname: draft-maldant-spice-smd-cwt-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Secure Patterns for Internet CrEdentials"
keyword:
  - cose
  - cwt
  - selective disclosure
venue:
  group: "Secure Patterns for Internet CrEdentials"
  type: "Working Group"
  mail: "spice@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/spice/"
  github: "beltram/draft-maldant-spice-smd-cwt"
  latest: "https://beltram.github.io/draft-maldant-spice-smd-cwt/draft-maldant-spice-smd-cwt.html"

author:
  - fullname: "Beltram Maldant"
    organization: SimpleLogin
    email: "beltram.ietf@pm.me"

normative:
  SD-CWT: I-D.draft-ietf-spice-sd-cwt-08
  RFC9162: CT

informative:

...

--- abstract

This document specifies an extension to Selective Disclosure CBOR Web
Tokens (SD-CWT) that reduces their size and improves their privacy
properties. In this extension, the individual hashes of redacted claims
are replaced with the root hash of a single Merkle tree, and disclosed
claims are presented with a Merkle proof.

--- middle

# Introduction {#intro}

This document defines Selective Merkle Disclosure CBOR Web Tokens (SMD-CWT), an extension to SD-CWT {{SD-CWT}}.

SD-CWT provides an efficient mechanism by which a Holder controls which
claims it presents to a Verifier, thereby improving the privacy of the
Holder. However, when only a subset of the claims is disclosed, the
Verifier still learns the total number of claims, which reduces the
privacy benefit of selective disclosure. Furthermore, each redacted claim
adds 34 bytes to the presented token (SD-KBT), even though the claim
itself is not disclosed to the Verifier.

To address both issues, this document specifies that, rather than hashing
each claim individually and placing the resulting hashes in a CBOR array,
the claims are placed in a Merkle tree and only the root hash of that tree
is included in the SD-CWT. A presentation of the token then includes only
a Merkle proof attesting to the integrity of the disclosed claims. The
size of this proof is logarithmic in the number of claims, whereas the
size of an enumeration of all redacted claims is linear in that number.

# Terminology

{::boilerplate bcp14-tagged}

The CBOR examples in this document are shown in CBOR Extended Diagnostic Notation (EDN) {{!I-D.ietf-cbor-edn-literals}}.

Data model constraints are described using the Concise Data Definition Language (CDDL) {{!RFC8610}}.

This document uses terms from CWT {{!RFC8392}}, COSE {{!RFC9052}} {{!RFC9053}},
and JWT {{!RFC7519}}.

The terms Claim Name, Claim Key, and Claim Value are defined in {{!RFC8392}}.

This document defines the following terms:

Selective Merkle Disclosure CBOR Web Token (SMD-CWT):
: An SD-CWT containing claims that can be selectively disclosed using a Merkle proof.

Selective Merkle Disclosure Key Binding Token (SMD-KBT):
: An SD-KBT used to demonstrate possession of a confirmation method and to present claims with a Merkle proof.

Copath:
: In a Merkle Tree for a given leaf, all the nodes alongside the inclusion proof of that leaf.

All other terms used in this document have the meaning defined in {{SD-CWT}}.

# Overview

SMD-CWT is an extension of SD-CWT: every SD-CWT is a valid SMD-CWT, but an SMD-CWT is not necessarily a valid
SD-CWT. The two differ only in the way redacted claims are authenticated and presented.

An SD-CWT Issuer enumerates the hashes of all claims in a CBOR array or map. In contrast, an SMD-CWT Issuer places
these hashes at the leaves of a left-balanced Merkle tree and replaces the set of claims in the original payload with
the root hash of that tree.

A Holder then selectively presents claims to a Verifier in an SMD-KBT by replacing the disclosed claims with proofs of
their inclusion in the Merkle tree. The root hash of that tree is authenticated by the Issuer in the SMD-CWT.

# Issuance {#smd-cwt-definition}

An SMD-CWT Issuer is also an SD-CWT Issuer: it MUST be capable of encoding and signing a regular SD-CWT when
requested and, in addition, supports the generation of SMD-CWTs.

{{fig-cwt-claimset-example}} shows an example CWT payload. Common claims are omitted for brevity.

~~~ cbor-diag
{
    "most_recent_inspection_passed": true,
    "inspector_license_number": "ABCD-123456",
    "inspection_dates" : [
        1549560720,   / 2019-02-07T17:32:00 /
        1612498440,   / 2021-02-04T20:14:00 /
        1674004740,   / 2023-01-17T17:19:00 /
    ],
    "inspection_location": {
        "country": "us",            / United States /
        "region": "ca",             / California /
        "postal_code": "94188"
    }
}
~~~
{: #fig-cwt-claimset-example title="CWT Claims Set"}

## Pre-Issuance {#tbr-tag}

An SMD-CWT Issuer MUST support the To Be Redacted Tag and the To Be Decoy Tag, both of which are defined in
{{Section 17.3.1 of SD-CWT}}.

The To Be Redacted Root Tag, defined in {{tbrrt}}, indicates to the Issuer that a claim or an array element is intended
to be part of a Merkle Disclosure.

{{fig-cwt-claimset-pre-issuance}} shows the example payload with the corresponding annotations.

~~~ cbor-diag
59([3, { /merkle-redacted of size 8/
    "most_recent_inspection_passed": true, /not-redacted/
    "inspector_license_number": 58("ABCD-123456"), /redacted/
    59([3, "inspection_dates"]) : [ /merkle-redacted of size 8/
        58(1549560720), /redacted/
        58(1612498440), /redacted/
        58(1674004740), /redacted/
    ],
    59([2, "inspection_location"]): { /merkle-redacted of size 4/
        "country": 58("us"),   /redacted/
        "region": 58("ca"),    /redacted/
        "postal_code": "94188" /not-redacted/
    }
}])
~~~
{: #fig-cwt-claimset-pre-issuance title="CWT Claims Set Pre Issuance"}

For each occurrence of the To Be Redacted Root Tag, at any nesting level, a Merkle tree is built, and all elements of
the tagged array or map are placed at its leaves.

## Merkle Tree {#merkle-tree}

This specification reuses binary Merkle trees as defined in {{Section 2.1.1 of !RFC9162}}. The arity of this Merkle Tree
is 2. A higher branching factor provides neither additional benefit nor the desired properties.

The Merkle tree is left-balanced and complete. Unbalanced trees are not used, as they are vulnerable to forgery attacks;
moreover, a left-balanced tree minimizes the number of required Decoys. {{fig-merkle-tree-example}} shows an example.
Also, a binary tree is easier to represent in memory and their indices are easier to compute as explained in {{Appendix
C of !RFC9420}}.

~~~ aasvg
              root
               |
           .---+---.
          /         \
   h(h(A)+h(B))    h(h(C)+h(d))
        / \         / \
       /   \       /   \
    h(A)   h(B)  h(C)  h(d)
      |     |     |     |
      |     |     |     |
      A     B     C    d

Node: 0  1  2  3  4  5  6

Leaf: 0     1     2     3
~~~
{: #fig-merkle-tree-example title="A 3-Claim Merkle Tree"}

### Placing Decoys

A Merkle tree MUST contain at least one Decoy. If the number of entries in the tree is a power of 2, the tree MUST be
grown.

Then, for each node in the Copath of the rightmost entry, the leftmost blank leaf of the subtree rooted at that node
MUST be replaced with a Decoy.

~~~ aasvg
                                                   X
                                                   |
                           .-----------------------+-----------------------.
                           |                                               |
                           V                                               W
                           |                                               |
                 .---------+---------.                           .---------+---------.
                /                     \                         /                     \
               R                       S                       T                       U
               |                       |                       |                       |
           .---+---.               .---+---.               .---+---.               .---+---.
          /         \             /         \             /         \             /         \
         J           K           L           M           N           O           P           Q
        / \         / \         / \         / \         / \         / \         / \         / \
       /   \       /   \       /   \       /   \       /   \       /   \       /   \       /   \
      A     B     C     D     E     F     G     H     I     d     d     _     d     _     _     _

Node: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

Leaf: 0     1     2     3     4     5     6     7     8     9     10    11    12    13    14    15
~~~
{: #fig-placing-decoys title="Placing Decoys"}

In {{fig-placing-decoys}}, the tree contains 9 entries. Decoys (`d`) are placed as follows:

1. The rightmost entry of the tree is `I`.
2. Its copath consists of nodes `18`, `21`, `27`, and `7` (`_`, `O`, `U`, and `V`).
3. For each node in this copath, the leftmost blank leaf of its subtree is replaced with a Decoy. A blank copath node
   that is itself a leaf, such as `18`, is replaced directly.
4. Consequently, nodes `18`, `20`, and `24` are replaced with Decoys.

## Computing the Merkle Root Hash {#merkle-root-hash}

The Issuer then computes the root hash of the Merkle tree as follows:

1. Each entry is encoded as a `bstr-encoded-merkle-salted`, as defined in {{cddl-salted-disclosed}}.
2. The resulting bytes are hashed using the hash algorithm indicated by `sd_alg`.
3. The hash of a parent node is the hash of its two children.

We define this operation as `MTH` for Merkle Tree Hash, similarly to the one described in {{Section 2.1.1 of !RFC9162}}.
Note that contrary to {{!RFC9162}}, we do not add a domain separation prefix to the leaves and the nodes for the simple
reason that our tree is balanced and its size is authenticated by the issuer. So each Inclusion Proof length must be the
same and so a second preimage resistance is not necessary as a node `MTH` cannot be used as a leaf one.

### Inserting the Merkle Root Hash in the payload

The root hash is then inserted into the original payload in place of the redacted claims. These claims are declared
within a container, which is either a map or an array. Since the container itself is annotated with the To Be Redacted
Root Tag, it cannot contain regular `redacted_claim_keys` (TODO: reference) or redacted array elements (tag 60) (TODO:
reference).

Regardless of the type of the parent container, the root hash is represented as a Redacted Root Hash: an array of two
elements, the first of which is the logarithmic size of the Merkle tree producing the root hash, and the second of
which is the root hash itself.

~~~ cddl
redacted_root_hash = [
  uint,  ; logarithm(2) of the number of leaves represented in this root hash
  bstr, ; root hash
]
~~~
{: #redacted-root-hash title="Redacted Root Hash"}

If the redaction occurs in a map, the root hash is placed in the `map_redacted_root_hash` claim as a Redacted Root Hash
({{redacted-root-hash}}).

~~~ cddl
/ map_redacted_root_hash / simple(60) : [ uint, bstr ]
~~~
{: #map-redacted-root-hash title="Map Redacted Root Hash"}

If the redaction occurs in an array, the root hash is placed in an `array_redacted_root_hash` element at the end of the
array (in the case of partial redaction).

~~~ cddl
"inspection_dates" : [
    1612498440, / item 1 not redacted /
    60([ uint, bstr ]) / root hash of item 0 and 2 /
],
~~~
{: #array-redacted-root-hash title="Array Redacted Root Hash"}

## Disclosure {#disclosure}

{{fig-smd-cwt-disclosures}} shows the disclosure of the `inspection_location` claim from
{{fig-cwt-claimset-example}}.

~~~ cbor-diag
<<[
    [
        /salt/   h'e39612407e99a347d61442957b3dfdac',
        /value/  "us",
        /claim/  "country"
    ],
    [
        /salt/   h'4569990258b25a5267c17a3e3a225ec3',
        /value/  "ca",
        /claim/  "region"
    ],
    [
        /salt/   h'e27403129cd88bf8fee4193e65339a35',
    ]
]>>
~~~
{: #fig-smd-cwt-disclosures title="SMD-CWT Disclosures"}

In an SMD-CWT, a Disclosure is an array of all redacted elements of a container (either a CBOR map or a CBOR array).
Each of these elements has the structure described in {{Section 3.2 of SD-CWT}}, except that it is not individually
wrapped in a CBOR bstr.

This Disclosure, along with all other Merkle Disclosures, is to be placed in the `sd_claims` unprotected header
parameter of the SMD-CWT, as defined in {{SD-CWT}}.

~~~ cddl
; an array of bstr-encoded Salted Disclosed Claims
{::include ./cddl/salted-array.cddl}
~~~
{: #cddl-salted-disclosed title="CDDL of Salted Disclosed Claims"}

## Content-Type

An SMD-CWT MUST declare its content type by including the `typ` protected header parameter {{!RFC9596}} with one of the
following values:

- the unsigned integer Constrained Application Protocol (CoAP) {{?RFC7252}} content-format value TBD1 (294 suggested);
  or
- the string content type value `application/smd-cwt+sd-cwt`.

# Presentation {#smd-kbt}

A Holder can selectively present some of the previously redacted claims in an SMD-KBT. An SMD-KBT has the same
structure as an SD-KBT. The only difference is that, rather than simply omitting the Disclosures it does not wish to
reveal, the Holder transforms each Disclosure introduced by this document from a simple enumeration of values into a
Disclosure Proof: an array containing one Merkle proof per presented leaf.

The Merkle proofs of a Disclosure Proof are not independent of each other. When a node of the copath of a presented
leaf is the root of a subtree containing another presented leaf, the Verifier recomputes the hash of that node from
the Merkle proof of that other leaf, so this hash is not included in the Disclosure Proof. Conversely, a hash that the
Verifier cannot recompute is included exactly once.

This section uses the following definitions:

Covered node:
: A node of the Merkle tree whose subtree contains at least one presented leaf. A presented leaf is itself covered.

Anchor:
: For a presented leaf, its smallest common ancestor with the presented leaves on its left, i.e., the lowest node
  whose subtree contains both that leaf and at least one presented leaf with a smaller node index. Since node indexes
  increase from left to right, the Anchor of a presented leaf is its smallest common ancestor with the presented leaf
  immediately preceding it. The leftmost presented leaf has no Anchor.

For a given Disclosure, the Holder proceeds as follows:

1. Iterate over the non-Decoy entries and select those to be presented. In the example of {{fig-presentation-tree}},
   leaves `A` to `E` are iterated over, and only `A`, `D`, and `E` are presented.
2. Compute the Anchor of each selected leaf. In the example of {{fig-presentation-tree}}, `A` has no Anchor, the Anchor
   of `D` is `R` (node 3), and the Anchor of `E` is `V` (node 7).
3. For each selected leaf, from left to right, build its Merkle proof as an array:
    1. Build a Merkle Disclosure from the node index of the leaf and the SMD-CWT entry, and place it as the first
       element of the Merkle proof.
    2. Walk the copath of that leaf, from the leaf up to the root. For each node of the copath:
        1. If the parent of that node is the Anchor of the leaf, stop: the node is a child of the Anchor, it is
           covered, and the Verifier recomputes its hash from the Merkle proof of a preceding presented leaf.
        2. Otherwise, if that node is covered, skip it and continue with the next node of the copath. Such a node is
           necessarily on the right of the leaf, and the Verifier recomputes its hash from the Merkle proof of a
           subsequent presented leaf.
        3. Otherwise, append the hash of that node to the Merkle proof. If that node is a leaf, its hash is the hash of
           its entry, computed as described in {{merkle-root-hash}}. Otherwise, its hash is the `MTH` of
           the subtree rooted at that node.
    3. The leftmost selected leaf has no Anchor: its walk ends after the last node of its copath, i.e., the child of
       the root.
4. The Disclosure Proof is the array of the Merkle proofs, ordered by increasing node index of their leaf.

Equivalently, the walk of step 3.2 stops at the first node of the copath that is covered and located on the left of the
leaf, and skips the nodes of the copath that are covered and located on the right of the leaf.

As a result, the only hashes included in a Disclosure Proof are those of the nodes that are not covered and whose
sibling is covered. Each of them is included exactly once, in the Merkle proof of the leftmost presented leaf of the
subtree rooted at its sibling. The more leaves are presented, the fewer hashes are required. In particular, when all
the non-Decoy entries are presented, the only hashes included are those of the nodes of the copath of the rightmost
entry that are located on its right, i.e., the subtrees containing only Decoys and blank leaves: at most n hashes,
where n is the size encoded in the `redacted_root_hash`. Apart from these hashes, such a Disclosure Proof only contains
the presented entries and their node indexes.

~~~ aasvg
                           V
                           |
                 .---------+---------.
                /                     \
               R                       S
               |                       |
           .---+---.               .---+---.
          /         \             /         \
         J           K           L           M
        / \         / \         / \         / \
       /   \       /   \       /   \       /   \
      A    (B)   (C)    D     E    d1     d2    _

Node: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14

Leaf: 0     1     2     3     4     5     6     7
~~~
{: #fig-presentation-tree title="Merkle Tree of a Presentation"}

In the following examples, `hash<<X>>` denotes the hash of the entry `X` when `X` is a leaf, and the `MTH` of the
subtree rooted at `X` otherwise.

`A` is the leftmost presented leaf and has no Anchor. Its copath consists of nodes 2 (`B`), 5 (`K`), and 11 (`S`). `B`
is not covered: its hash is appended. `K` is covered by `D` and `S` is covered by `E`: both are skipped. The Merkle
proof for `A` is therefore:

~~~ cbor-diag
[
  [0, h'4e2f040d5bb57daf86698066be470c24', "A"],
  hash<<B>>,
]
~~~
{: #fig-merkle-proof-leaf-a title="Merkle Proof of A"}

The copath of `D` consists of nodes 4 (`C`), 1 (`J`), and 11 (`S`). `C` is not covered: its hash is appended. The
parent of `J` is `R`, the Anchor of `D`: the walk stops. The Merkle proof for `D` is therefore:

~~~ cbor-diag
[
  [6, h'add7fe2b052af5faf0ec67f57eb1454f', "D"],
  hash<<C>>,
]
~~~
{: #fig-merkle-proof-leaf-d title="Merkle Proof of D"}

The copath of `E` consists of nodes 10 (`d1`), 13 (`M`), and 3 (`R`). Neither `d1` nor `M` is covered: their hashes are
appended. The parent of `R` is `V`, the Anchor of `E`: the walk stops. The Merkle proof for `E` is therefore:

~~~ cbor-diag
[
  [8, h'6e0e0494c6aa9f9072c3410304aab730', "E"],
  hash<<d1>>,
  hash<<M>>,
]
~~~
{: #fig-merkle-proof-leaf-e title="Merkle Proof of E"}

The resulting Disclosure Proof is shown in {{fig-disclosure-proof}}. It contains 4 hashes. The hashes of `J`, `K`, `R`,
`L`, `S`, and `V` are recomputed by the Verifier.

~~~ cbor-diag
[
  [ [0, h'4e2f040d5bb57daf86698066be470c24', "A"], hash<<B>> ],             / "A" /
  [ [6, h'add7fe2b052af5faf0ec67f57eb1454f', "D"], hash<<C>> ],             / "D" /
  [ [8, h'6e0e0494c6aa9f9072c3410304aab730', "E"], hash<<d1>>, hash<<M>> ]  / "E" /
]
~~~
{: #fig-disclosure-proof title="Disclosure Proof"}

If the Holder presents all the non-Decoy entries `A` to `E` of {{fig-presentation-tree}} instead, the Merkle proofs of
`A`, `B`, `C`, and `D` each contain only their Merkle Disclosure, since every node of their copath is either covered
on their right or a child of their Anchor. The Merkle proof of `E` is unchanged and contains `hash<<d1>>` and
`hash<<M>>`. The resulting Disclosure Proof contains 2 hashes.

# Verification

An SMD-CWT Verifier MUST perform the same verification as an SD-CWT Verifier. This section only describes how to
validate a Disclosure Proof against the `redacted_root_hash` of the corresponding container. In this section, n is the
size encoded in the `redacted_root_hash`; the Merkle tree therefore has 2^n leaves, its node indexes range from 0 to
2^(n+1) - 2, and its root has node index 2^n - 1.

The Verifier proceeds as follows:

1. Verify that the Disclosure Proof contains at least one and at most 2^n - 1 Merkle proofs.
2. For each Merkle proof:
    1. Verify that it contains at most n + 1 elements.
    2. Verify that its first element is a Merkle Disclosure, and that the node index of this Merkle Disclosure is the
       node index of a leaf, i.e., an even number less than or equal to 2^(n+1) - 2.
3. Verify that the node indexes of the Merkle Disclosures are strictly increasing from the first Merkle proof to the
   last one.
4. From these node indexes, determine the covered nodes and the Anchor of each presented leaf, as defined in
   {{smd-kbt}}.
5. Maintain a table associating node indexes with hashes, initially empty.
6. Process the Merkle proofs from the last one to the first one, i.e., from the rightmost presented leaf to the
   leftmost one. For each Merkle proof:
    1. Remove the node index from the Merkle Disclosure. The remaining array is an SMD-CWT entry: compute its hash as
       described in {{merkle-root-hash}}. This is the current hash, and the leaf is the current node.
    2. Walk the copath of the leaf, from the leaf up to the root. For each node of the copath:
        1. If the parent of that node is the Anchor of the leaf, record the current hash for the current node in the
           table, and stop the walk.
        2. Otherwise, if that node is covered, it is located on the right of the leaf: take its hash from the table.
           This hash has necessarily been recorded while processing a Merkle proof located after the current one.
        3. Otherwise, take the next hash of the Merkle proof. If the Merkle proof has no hash left, the validation
           fails.
        4. Hash the concatenation of the current hash and the hash obtained in the previous step, the hash of the left
           child coming first: if the node index of the current node is smaller than the node index of the copath
           node, the current hash comes first; otherwise, it comes second. The result becomes the current hash, and
           the parent of the current node becomes the current node.
    3. Verify that all the hashes of the Merkle proof have been consumed. If not, the validation fails.
7. The walk of the first Merkle proof, whose leaf has no Anchor, ends at the root. Verify that its final current hash is
   equal to the root hash in the `redacted_root_hash`.
8. If any of these verifications fails, the validation of the whole Disclosure Proof fails.

Applied to the Disclosure Proof of {{fig-disclosure-proof}}, the Verifier determines that the Anchor of `D` is `R` and
the Anchor of `E` is `V`, then:

1. Processes the Merkle proof of `E`: it combines `hash<<E>>` with `hash<<d1>>` to obtain `hash<<L>>`, then with
   `hash<<M>>` to obtain `hash<<S>>`. The next copath node, `R`, is a child of `V`, the Anchor of `E`: `hash<<S>>` is
   recorded for node 11.
2. Processes the Merkle proof of `D`: it combines `hash<<C>>` with `hash<<D>>` to obtain `hash<<K>>`. The next copath
   node, `J`, is a child of `R`, the Anchor of `D`: `hash<<K>>` is recorded for node 5.
3. Processes the Merkle proof of `A`: it combines `hash<<A>>` with `hash<<B>>` to obtain `hash<<J>>`, then with
   `hash<<K>>` taken from the table to obtain `hash<<R>>`, then with `hash<<S>>` taken from the table to obtain
   `hash<<V>>`.
4. Verifies that `hash<<V>>` is equal to the root hash in the `redacted_root_hash`.

# Benefits over SD-CWT

## Smaller Tokens

TODO: Logarithmically smaller.

## Privacy

TODO: Does not leak the array size; does not leak the map structure.

## Free Decoys

In SD-CWT, each Decoy adds one hash, i.e., 34 bytes, to the presented token.

In SMD-CWT, the presented token contains only the root hash and the Merkle proofs of the disclosed claims. The size of
these proofs depends on the height of the tree, not on the number of its leaves. Decoys can therefore be added at no
cost up to the next power of two. Beyond that, doubling the number of leaves adds one level to the tree, which costs
only one additional hash, i.e., 34 bytes: the new copath node is not covered and is therefore included only once in
the Disclosure Proof, in the Merkle proof of the leftmost presented leaf (see {{smd-kbt}}).

## Verification Time

TODO: The sparser the presentation, the faster the verification; however, a full presentation takes twice as long to
verify.

# Privacy Considerations {#privacy-considerations}

## Leakage of Claim Order

# Security Considerations {#security}

TODO Security

# IANA Considerations {#iana}

## CBOR Tags

IANA is requested to register the following tag in the
["CBOR Tags" registry](https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml#tags):

### To Be Redacted Root Tag {#tbrrt}

This tag identifies an array or map containing elements to be redacted. The value of this tag is an array of two
elements: the first is the logarithmic size of the desired proof length, and the second is the wrapped array or map.

* Tag: TBD2 (suggested 59)
* Data Item: (array or map)
* Semantics: An array or map containing elements to be redacted with a Merkle proof, each tagged with the To Be
  Redacted Tag defined in {{Section 17.3.1 of SD-CWT}}.
* Specification Document (s): {{tbr-tag}} of this specification

## Media Types

IANA is requested to register the following media types in the
["Media Types" registry](https://www.iana.org/assignments/media-types/media-types.xhtml#application):

### application/smd-cwt+sd-cwt

The following completed registration template is provided:

* Type name: application
* Subtype name: smd-cwt
* Required parameters: n/a
* Optional parameters: n/a
* Encoding considerations: binary
* Security considerations: {{security}} of this specification and {{!RFC8392}}
* Interoperability considerations: n/a
* Published specification: {{smd-cwt-definition}} of this specification
* Applications that use this media type: TBD
* Fragment identifier considerations: n/a
* Additional information:
    - Magic number (s): n/a
    - File extension (s): n/a
    - Macintosh file type code (s): n/a
* Person & email address to contact for further information:
  SPICE WG mailing list (spice@ietf.org) or
  IETF Security Area (saag@ietf.org)
* Intended usage: COMMON
* Restrictions on usage: none
* Author: See Author's Addresses section
* Change controller: IETF
* Provisional registration? No

### application/smd-kbt+sd-kbt

The following completed registration template is provided:

* Type name: application
* Subtype name: kb+cwt
* Required parameters: n/a
* Optional parameters: n/a
* Encoding considerations: binary
* Security considerations: {{security}} of this specification and {{!RFC8392}}
* Interoperability considerations: n/a
* Published specification: {{smd-kbt}} of this specification
* Applications that use this media type: TBD
* Fragment identifier considerations: n/a
* Additional information:
    - Magic number (s): n/a
    - File extension (s): n/a
    - Macintosh file type code (s): n/a
* Person & email address to contact for further information:
  SPICE WG mailing list (spice@ietf.org) or
  IETF Security Area (saag@ietf.org)
* Intended usage: COMMON
* Restrictions on usage: none
* Author: See Author's Addresses section
* Change controller: IETF
* Provisional registration? No

--- back

# Complete CDDL Schema {#cddl}

~~~~~~~~~~ cddl
{::include ./cddl/preissued.cddl}
{::include ./cddl/salted-array.cddl}
{::include ./cddl/redacted-simple-tag.cddl}
~~~~~~~~~~
{: #cddl-schema title="A complete CDDL description of SMD-CWT"}

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
