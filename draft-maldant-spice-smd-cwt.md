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
  I-D.draft-maldant-spice-smd-cwt: SD-CWT

informative:

...

--- abstract

This document specifies an extension to Selective Disclosure CBOR Web
Tokens (SD-CWT) that reduces their size and improves their privacy
properties. The individual hashes of redacted claims are replaced by a
single Merkle tree root, and disclosed claims are presented using a Merkle
proof.

--- middle

# Introduction {#intro}

This document defines Selective Merkle Disclosure CBOR Web
Tokens (SMD-CWT), an extension to {{SD-CWT}}.

SD-CWT provides an efficient mechanism for a Holder to control
which claims it presents to a Verifier, thereby improving its privacy.
However, when only a subset of the claims is disclosed, the Verifier
still learns the total number of claims, which reduces the privacy
benefit of selective disclosure. In addition, each redacted claim, even
though it is not presented to the Verifier, adds 34 bytes to the
presented token (SD-KBT).

To address both issues, this document specifies that, instead of hashing
each claim individually and placing the hashes in a CBOR array, the claims
are placed in a Merkle tree and only the root hash is included in the
SD-CWT. A presentation of the token then only needs to include a Merkle
proof for the disclosed claim(s) to attest to their integrity. The size
of this proof is logarithmic in the number of claims, whereas enumerating
all redacted claims is linear.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
