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
title: "Selective Merkle Disclosure CBOR Web Tokens (SMD-CWT)"
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
properties. In this extension, the individual hashes of the redacted
claims of a map or an array are replaced with the root hash of a Merkle
tree, and disclosed claims are presented with a Merkle proof.

--- middle

# Introduction {#intro}

This document defines Selective Merkle Disclosure CBOR Web Tokens (SMD-CWT), an extension to SD-CWT {{SD-CWT}}.

SD-CWT provides an efficient mechanism by which a Holder controls which
claims it presents to a Verifier, thereby improving the privacy of the
Holder. However, when the presentation is sparse and only a subset of the claims is disclosed, a
Verifier learns the total number of claims, reducing the
privacy benefit of selective disclosure. Furthermore, each redacted claim
adds 34 bytes to the presented token (SD-KBT), even though the claim
itself is not disclosed to the Verifier.

To address both issues, this document specifies that, rather than hashing
each claim individually and placing the resulting hashes in a CBOR array,
the claims of each map or array are placed in a Merkle tree and only the root hash of that tree
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
: For a given leaf of a Merkle tree, the siblings of the nodes on the path from that leaf to the root.

All other terms used in this document have the meaning defined in {{SD-CWT}}.

# Overview

SMD-CWT is an extension of SD-CWT: every SD-CWT is a valid SMD-CWT, but an SMD-CWT is not necessarily a valid SD-CWT.
The two differ only in the way redacted claims are authenticated and presented.

An SD-CWT Issuer enumerates the hashes of all redacted claims, either in the `redacted_claim_keys` array of a map, or in
place of the redacted elements of an array. In contrast, an SMD-CWT Issuer places these hashes at the leaves of a
left-balanced Merkle tree and replaces the set of claims in the original payload with the root hash of that tree.

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

An SMD-CWT Issuer MUST support the `To Be Redacted Tag` and the `To Be Decoy Tag`, both of which are defined in
{{Section 17.3.1 of SD-CWT}}.

The `To Be Redacted Root Tag` (tag 59), defined in {{tbrrt}}, indicates to the Issuer that a claim or an array element
is intended to be selectively disclosed with a Disclosure Proof. This tag wraps the element to be redacted in an array
of size 2. The first element of this array is the base-2 logarithm of the size of the Merkle tree used for redaction.
This size MAY be
greater than the number of claims to redact, but MUST NOT be lower.

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

{: #fig-cwt-claimset-pre-issuance title="CWT Claims Set Before Issuance"}

For each occurrence of the To Be Redacted Root Tag, at any nesting level, a Merkle tree is built, and all elements of
the tagged array or map are placed at its leaves.

## Merkle Tree {#merkle-tree}

This specification uses binary Merkle trees similar to those defined in {{Section 2.1.1 of !RFC9162}}, with one
difference: no domain separation prefix is added to the leaves and the nodes (see {{merkle-root-hash}}). A higher
branching factor would provide neither additional benefit nor the desired properties.

The Merkle tree is a left-balanced perfect binary tree, as shown in {{fig-merkle-tree-example}}. Unbalanced trees are
not used, as they are vulnerable to forgery attacks. Perfect binary trees are also easier to represent in memory, and
their node indexes are easier to compute, as explained in {{Appendix C of !RFC9420}}.

~~~ aasvg
              root
               |
           .---+---.
          /         \
   h(h(A)||h(B))    h(h(C)||h(d))
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

In {{fig-cwt-claimset-pre-issuance}}, the redaction of the "inspection_location" claim places the "country" claim at
node index 0 and the "region" claim at node index 2. Since a size of 2 is requested, the tree has a height of 2 and its
last node index is 6, as in {{fig-merkle-tree-example}}.

### Decoys {#decoys}

Decoys serve a different purpose than in SD-CWT: they do not directly hinder the Verifier's inference of private
information. Instead, they act as a blinding factor that prevents a Verifier from inferring the number of leaves, since
the hash of a blank leaf is the hash of zero bytes. Decoys are therefore mandatory.

Replacing every blank leaf with a Decoy would needlessly inflate the SMD-CWT. Instead, for each node in the Copath of
the
rightmost entry, the leftmost blank leaf of the subtree rooted at that node MUST be replaced with a Decoy. When the
entries fill all the leaves of the tree, no Decoy is added.

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

1. Each entry is encoded as a `bstr-encoded-salted-merkle-entry`, as defined in {{cddl-salted-array}}.
2. The resulting bytes are hashed using the hash algorithm indicated by `sd_alg` in the protected header.
3. The hash of a parent node is the hash of the concatenation of the hashes of its two children, the hash of the left
   child coming first.

This operation is denoted `MTH` (Merkle Tree Hash), similarly to the one described in {{Section 2.1.1 of !RFC9162}}.
Unlike in {{!RFC9162}}, no domain separation prefix is added to the leaves and the nodes: since the tree is balanced and
its size is authenticated by the Issuer, all leaves are at the same depth. The `MTH` of an interior node therefore
cannot be used as the hash of a leaf, and second preimage resistance between leaves and nodes is not required.

### Inserting the Merkle Root Hash in the Payload

The root hash is then inserted into the original payload in place of the redacted claims. These claims are declared
within a container, which is either a map or an array. Since the container itself is annotated with the `To Be Redacted
Root Tag`, it cannot contain regular `redacted_claim_keys` (simple value 59) or redacted array elements (tag 60).

Regardless of the type of the parent container, the root hash is represented as a `redacted_root_hash`: an array of two
elements, the first of which is the logarithmic size of the Merkle tree producing the root hash, and the second of
which is the root hash itself.

~~~ cddl
redacted_root_hash = [
  uint,  ; base-2 logarithm of the number of leaves represented in this root hash
  bstr, ; root hash
]
~~~

{: #redacted-root-hash title="Redacted Root Hash"}

If the redaction occurs in a map, the `redacted_root_hash` is placed in the `redacted_claim_hash` claim.

~~~ cddl
/ redacted_claim_hash / simple(60) : [ uint, bstr ] / redacted_root_hash /
~~~

{: #redacted-claim-hash title="Redacted Claim Hash"}

If the redaction occurs in an array, the root hash is placed in an `redacted_element_hash` element appended to the
array, after the elements that are not redacted.

~~~ cddl
"inspection_dates" : [
    1612498440, / item 1 not redacted /
    89([ uint, bstr ]) / root hash of item 0 and 2 /
],
~~~

{: #redacted-element-hash title="Redacted Element Hash"}

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

In an SMD-CWT, a Disclosure is an array of all redacted elements of a given container (either a CBOR map or array).
Each of these elements has the structure described in {{Section 3.2 of SD-CWT}}, except that they are not individually
wrapped in a CBOR bstr (but the outer disclosure is).

This Disclosure, along with all other Disclosures, MUST be placed in the `sd_claims` unprotected header
parameter of the SMD-CWT, as defined in {{SD-CWT}}.

~~~ cddl
; an array of bstr-encoded Salted Disclosed Claims
{::include ./cddl/salted-array.cddl}
~~~

{: #cddl-salted-array title="CDDL of Salted Disclosed Claims"}

{{fig-issued-smd-cwt}} shows the SMD-CWT issued from {{fig-cwt-claimset-pre-issuance}}. Each of the three containers
tagged with the To Be Redacted Root Tag is replaced with a Redacted Root Hash, and its Disclosure is placed in
`sd_claims`:

~~~ cbor-diag
{::include ./examples/issuer_cwt.edn}
~~~

{: #fig-issued-smd-cwt title="Issued SMD-CWT with all disclosures"}

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

The Merkle proofs of a Disclosure Proof are not independent of each other. The hash of a copath node is not included
in a Merkle proof when its subtree contains a presented leaf, since the Verifier recomputes it, or when it is already
included in the Merkle proof of a previous leaf.

For a given Disclosure, the Holder proceeds as follows:

1. Iterate over the non-Decoy entries and select those to be presented. In the example of {{fig-presentation-tree}},
   leaves `A` to `E` are iterated over, and only `A`, `D`, and `E` are presented.
2. Initialize a visited set with the node indexes of the selected leaves and of all their ancestors. In the example of
   {{fig-presentation-tree}}, the visited set contains nodes 0, 6, and 8 (`A`, `D`, and `E`) and nodes 1, 3, 5, 7, 9,
   and 11 (`J`, `R`, `K`, `V`, `L`, and `S`).
3. For each selected leaf, from left to right, build its Merkle proof as an array:
    1. Build a Redacted Entry by prefixing the SMD-CWT entry with the node index of the leaf, and place it as the
       first element of the Merkle proof.
    2. Walk the copath of that leaf, from the leaf up to the root. For each node of the copath:
        1. If this node is in the visited set, skip it.
        2. Otherwise, append the hash of that node to the Merkle proof and add the node to the visited set. If that
           node is a leaf, its hash is the hash of its entry, computed as described in {{merkle-root-hash}}.
           Otherwise, its hash is the `MTH` of the subtree rooted at that node.
4. The Disclosure Proof is the array of the Merkle proofs, ordered by increasing node index of their leaf.

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

The copath of `A` consists of nodes 2 (`B`), 5 (`K`), and 11 (`S`). `B` is not visited: its hash is appended. `K` and
`S` are visited: they are skipped. The Merkle proof for `A` is therefore:

~~~ cbor-diag
[
  [0, h'4e2f040d5bb57daf86698066be470c24', "A"],
  hash<<B>>,
]
~~~

{: #fig-merkle-proof-leaf-a title="Merkle Proof of A"}

The copath of `D` consists of nodes 4 (`C`), 1 (`J`), and 11 (`S`). `C` is not visited: its hash is appended. `J` and
`S` are visited: they are skipped. The Merkle proof for `D` is therefore:

~~~ cbor-diag
[
  [6, h'add7fe2b052af5faf0ec67f57eb1454f', "D"],
  hash<<C>>,
]
~~~

{: #fig-merkle-proof-leaf-d title="Merkle Proof of D"}

The copath of `E` consists of nodes 10 (`d1`), 13 (`M`), and 3 (`R`). `d1` and `M` are not visited: their hashes are
appended. `R` is visited: it is skipped. The Merkle proof for `E` is therefore:

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

If the Holder presents all the non-Decoy entries `A` to `E` of {{fig-presentation-tree}} instead, every copath node of
`A`, `B`, `C`, and `D` is visited: their Merkle proofs each contain only their Redacted Entry. The Merkle proof of
`E` is unchanged and contains `hash<<d1>>` and `hash<<M>>`. The resulting Disclosure Proof contains 2 hashes.

If the Holder presents only `A` and `B`, the Merkle proof of `A` contains `hash<<K>>` and `hash<<S>>`, which are then
visited, so the Merkle proof of `B` contains no hash.

# Verification

An SMD-CWT Verifier MUST perform the same verification as an SD-CWT Verifier. This section only describes how to
validate a Disclosure Proof against the `redacted_root_hash` of the corresponding container.

To validate an SMD-KBT, repeat this verification for all the disclosures presented. Then also perform the validation
described in {{Section 9 of SD-CWT}} since they all should pass as well.

In this section, `n` is the size encoded in the `redacted_root_hash` (the first element of the array); the Merkle tree
therefore has `2^n` leaves, its node indexes range from 0 to `2^ (n+1) - 2`, and its root has node index `2^n - 1`.

The Verifier proceeds as follows:

1. Verify that the Disclosure Proof contains at least one and at most `2^n` Merkle proofs.
2. For each Merkle proof:
    1. Verify that it contains at most n + 1 elements.
    2. Verify that its first element is a Redacted Entry, and that the node index of this Redacted Entry is the
       node index of a leaf, i.e., an even number less than or equal to 2^ (n+1) - 2.
    3. Verify that this Redacted Entry matches the type of the container: a `salted-merkle-claim` if the container
       is a map, or a `salted-merkle-element` if it is an array, as defined in {{cddl-salted-array}}. In
       particular, a Redacted Entry of a Decoy, which only contains a salt, is rejected.
3. Verify that the node indexes of the Redacted Entries are strictly increasing from the first Merkle proof to the
   last one.
4. Initialize a visited set with the node indexes of the presented leaves and of all their ancestors.
5. Maintain a table associating node indexes with hashes, initially empty.
6. For each Merkle proof:
    1. Remove the node index from the Redacted Entry. The remaining array is an SMD-CWT entry: compute its hash as
       described in {{merkle-root-hash}} and record it for the leaf in the table.
    2. Walk the copath of the leaf, from the leaf up to the root. For each node of the copath:
        1. If this node is in the visited set, skip it.
        2. Otherwise, compute the hash of the Merkle proof, record it for that node in the table, and add the node to
           the visited set. If the Merkle proof has no hash left, the validation fails.
7. While the table contains two nodes that are siblings, remove both of them from the table, hash the concatenation
   of their hashes, the hash of the left sibling coming first, and record the result for their parent in the table.
8. Verify that the table contains a single node, that this node is the root, and that its hash is equal to the root
   hash in the `redacted_root_hash`.
9. If any of these verifications fails, the validation of the whole Disclosure Proof fails.

Applied to the Disclosure Proof of {{fig-disclosure-proof}}, the visited set initially contains nodes 0, 1, 3, 5, 6, 7,
8, 9, and 11. The Verifier then:

1. Processes the Merkle proof of `A`: it records `hash<<A>>` for node 0 and `hash<<B>>` for node 2. Nodes 5 and 11
   are visited.
2. Processes the Merkle proof of `D`: it records `hash<<D>>` for node 6 and `hash<<C>>` for node 4. Nodes 1 and 11 are
   visited.
3. Processes the Merkle proof of `E`: it records `hash<<E>>` for node 8, `hash<<d1>>` for node 10, and `hash<<M>>` for
   node 13. Node 3 is visited.
4. Combines the siblings `A` and `B` into `hash<<J>>` (node 1), `C` and `D` into `hash<<K>>` (node 5), `E` and `d1`
   into `hash<<L>>` (node 9), then `J` and `K` into `hash<<R>>` (node 3), `L` and `M` into `hash<<S>>` (node 11), and
   finally `R` and `S` into `hash<<V>>` (node 7).
5. Verifies that the table only contains node 7, the root, and that `hash<<V>>` is equal to the root hash in the
   `redacted_root_hash`.

# Benefits over SD-CWT

## Smaller Tokens

In SD-CWT, the payload contains the hash of every redacted entry, disclosed or not: a container with `e` redacted
entries
costs `34 * e` bytes in every presentation (assuming `sd_alg` is SHA-256).

The size of a presentation therefore grows linearly with e in SD-CWT, but only logarithmically in SMD-CWT. For
instance, the Disclosure Proof of {{fig-disclosure-proof}} contains 4 hashes in addition to the root hash, i.e.,
34 * 5 = 170 bytes, whereas an SD-CWT with the same 5 entries and 2 Decoys contains 34 * 7 = 238 bytes of hashes.

## Privacy

In SD-CWT, the payload contains one hash per redacted entry, so a Verifier learns the number of redacted entries of each
container Also given presentation of SD-KBT are linkable, a Verifier can witness an array of redacted elements grow or
shrink which degrades privacy of the Holder. Decoys can be used to mitigate that, but they are expensive in terms of
space wasted.

In SMD-CWT, the Verifier only learns the logarithmic size of the tree. The Issuer can choose a larger n to hide this
number further, at the cost of at most one hash per additional level (see {{free-decoys}}).

## Free Decoys {#free-decoys}

In SD-CWT, each Decoy adds one hash, i.e., 34 bytes, to the presented token.

In SMD-CWT, the presented token contains only the root hash and the Merkle proofs of the disclosed claims. The size of
these proofs depends on the height of the tree, not on the number of its leaves. Decoys can therefore be added at no
cost up to the next power of two. Beyond that, doubling the number of leaves adds one level to the tree, which costs
at most one additional hash, i.e., 34 bytes: the subtree rooted at the new copath node contains no presented leaf, and
its hash is included at most once in the Disclosure Proof (see {{smd-kbt}}).

## Verification Time

TODO: The sparser the presentation, the faster the verification; however, a full presentation takes twice as long to
verify.

# Privacy Considerations {#privacy-considerations}

## Leakage of Claim Order {#claim-order}

# Security Considerations {#security}

TODO Security

# IANA Considerations {#iana}

## CBOR Tags

IANA is requested to register the following tag in the
["CBOR Tags" registry](https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml#tags):

### To Be Redacted Root Tag {#tbrrt}

This tag identifies an array or map containing elements to be redacted. The value of this tag is an array of two
elements: the first is the base-2 logarithm of the number of leaves of the Merkle tree, i.e., the maximum number of
hashes in a
Merkle proof, and the second is the wrapped array or map.

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
* Subtype name: smd-kbt
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
