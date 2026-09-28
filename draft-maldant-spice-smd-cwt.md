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

informative:

...

--- abstract

This document specifies an extension to Selective Disclosure CBOR Web
Tokens (SD-CWT) that reduces their size and improves their privacy
properties. The individual hashes of redacted claims are replaced by the
root of a single Merkle tree, and disclosed claims are presented with a
Merkle proof.

--- middle

# Introduction {#intro}

This document defines Selective Merkle Disclosure CBOR Web
Tokens (SMD-CWT), an extension to SD-CWT {{SD-CWT}}.

SD-CWT provides an efficient mechanism for a Holder to control
which claims it presents to a Verifier, thereby improving the Holder's
privacy.
However, when only a subset of the claims is disclosed, the Verifier
still learns the total number of claims, which reduces the privacy
benefit of selective disclosure. In addition, each redacted claim adds
34 bytes to the presented token (SD-KBT), even though the claim itself is
not disclosed to the Verifier.

To address both issues, this document specifies that, instead of hashing
each claim individually and placing the hashes in a CBOR array, the claims
are placed in a Merkle tree and only its root hash is included in the
SD-CWT. A presentation of the token then only includes a Merkle proof
attesting to the integrity of the disclosed claims. The size of
this proof is logarithmic in the number of claims, whereas the size of an
enumeration of all redacted claims is linear.

# Terminology

{::boilerplate bcp14-tagged}

The CBOR examples in this document are shown in CBOR Extended Diagnostic Notation (EDN) {{!I-D.ietf-cbor-edn-literals}}.

Data model constraints are described using the Concise Data Definition Language (CDDL) {{!RFC8610}}.

This specification uses terms from CWT {{!RFC8392}}, COSE {{!RFC9052}} {{!RFC9053}},
and JWT {{!RFC7519}}.

The terms Claim Name, Claim Key, and Claim Value are defined in {{!RFC8392}}.

This specification defines the following new terms:

Selective Merkle Disclosure CBOR Web Token (SMD-CWT):
: An SD-CWT with claims enabling selective disclosure using a Merkle proof.

Selective Merkle Disclosure Key Binding Token (SMD-KBT):
: An SD-KBT used to demonstrate possession of a confirmation method and to present claims with a Merkle proof.

All other terms used in this document have the same meaning as in {{SD-CWT}}.

# Overview

SMD-CWT is an extension of SD-CWT: every SD-CWT is a valid SMD-CWT, but an SMD-CWT is not necessarily a valid
SD-CWT. The two differ only in how redacted claims are authenticated and presented.

Whereas an SD-CWT Issuer enumerates all claim hashes in a CBOR array or map, an SMD-CWT Issuer places these hashes at
the leaves of a left-balanced Merkle tree and replaces the set of claims in the original payload with the root hash of
that tree.

A Holder then selectively presents claims to a Verifier in an SMD-KBT by replacing the disclosed claims with proofs of
their inclusion in the Merkle tree. The root hash of that tree is authenticated by the Issuer in the SMD-CWT.

# Issuance {#smd-cwt-definition}

An SMD-CWT Issuer is also an SD-CWT Issuer: it MUST be capable of encoding and signing a regular SD-CWT when
requested, and additionally supports generating SMD-CWTs.

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

An SMD-CWT Issuer MUST support the To Be Redacted Tag and the To Be Decoy Tag, both defined in
{{Section 17.3.1 of SD-CWT}}.

The To Be Redacted Root Tag, defined in {{tbrrt}}, signals to the Issuer that a claim or array element is intended to be
part of a Merkle Disclosure.

{{fig-cwt-claimset-pre-issuance}} shows the example payload annotated accordingly.

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

For each occurrence of the To Be Redacted Root Tag, at any level, a Merkle tree is built and all elements of the tagged
array or map are placed at its leaves.

## Merkle Tree {#merkle-tree}

This specification is restricted to binary Merkle trees, as a higher branching factor yields neither additional
benefit nor the desired properties.

The Merkle tree is left-balanced. Unbalanced trees are avoided because they are vulnerable to forgery attacks, and a
left-balanced tree minimizes the number of Decoys required. {{fig-merkle-tree-example}} shows an example.

~~~ aasvg
              root
               |
           .---+---.
          /         \
   h(h(A)+h(B))    h(h(C)+h({D}))
        / \         / \
       /   \       /   \
    h(A)   h(B)  h(C)  h({D})
      |     |     |     |
      |     |     |     |
      A     B     C    {D}

Node: 0  1  2  3  4  5  6

Leaf: 0     1     2     3
~~~
{: #fig-merkle-tree-example title="A 3-Claim Merkle Tree"}

### Placing Decoys

A Merkle tree MUST contain at least one Decoy. If the number of entries in the tree is a power of 2, the tree MUST be
grown.

Then, for each node in the copath of the rightmost entry, the leftmost blank leaf of the subtree rooted at that node
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
2. Its copath is `18, 21, 27, 7` (`_, O, U, V`).
3. For each node on this copath, the leftmost blank leaf of its subtree is replaced with a Decoy. A blank copath node
   that is itself a leaf, such as `18`, is replaced directly.
4. Hence, nodes `18`, `20`, and `24` are replaced with Decoys.

## Computing the Merkle Root

The Issuer then computes the root hash of the Merkle tree as follows:

1. Each entry is encoded as a `bstr-encoded-merkle-salted`, as defined in {{cddl-salted-disclosed}}.
2. The resulting bytes are hashed with the `sd_alg` hash algorithm.
3. The hash of a parent node is the hash of its two children.

The root hash is placed in the `redacted claim keys`, in a Merkle Root value ({{merkle-root-value}}). A Merkle Root value
is an array of two elements: the first is the logarithmic size of the Merkle tree producing the root hash, and the
second is the root hash.

~~~ cddl
merkle_root_value = [
  uint  ; logarithm(2) of the number of leaves represented in this root hash
  bstr, ; root hash
]
~~~
{: #merkle-root-value title="Merkle Root value"}

## Disclosure

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

In an SMD-CWT, a disclosure is an array of all the redacted elements of a container (either a CBOR map or array).
These elements have the structure described in {{Section 3.2 of SD-CWT}}, except that they are not individually wrapped
in a CBOR bstr.

This disclosure, along with all other Merkle disclosures, has to be placed in the `sd_claims` unprotected header
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

# Privacy Considerations {#privacy}

# Security Considerations {#security}

TODO Security

# IANA Considerations {#iana}

## CBOR Tags

IANA is requested to add the following entry to the
["CBOR Tags" registry](https://www.iana.org/assignments/cbor-tags/cbor-tags.xhtml#tags):

### To Be Redacted Root Tag {#tbrrt}

This tag identifies an array or map containing elements to be redacted. The value of this tag is an array of two
elements: the first is the logarithmic size of the desired proof length, and the second is the wrapped array or map.

* Tag: TBD2 (suggested 59)
* Data Item: (array or map)
* Semantics: An array or map containing elements to be redacted with a Merkle proof, each tagged with the To Be
  Redacted Tag defined in {{Section 17.3.1 of SD-CWT}}.
* Specification Document(s): {{tbr-tag}} of this specification

## Media Types

IANA is requested to add the following entries to
the ["Media Types" registry](https://www.iana.org/assignments/media-types/media-types.xhtml#application):

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
    - Magic number(s): n/a
    - File extension(s): n/a
    - Macintosh file type code(s): n/a
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
    - Magic number(s): n/a
    - File extension(s): n/a
    - Macintosh file type code(s): n/a
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
{: #cddl-schema title="A complete CDDL description of SD-CWT"}

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
