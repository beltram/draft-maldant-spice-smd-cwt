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

This document specifies an extension of Selective Disclosure CBOR Web Tokens (SD-CWT) aiming at reducing their size and improving their privacy.
It proposes to aggregate the individual hashes of the redacted claims by a single Merkle proof.

--- middle

# Introduction {#intro}

This specification creates a new format for Selective Disclosure CBOR Web Token (SD-CWT) specification {{SD-CWT}}.
Selective Disclosure CBOR Web tokens provide an efficient way for a Relying Party to get more control on the claims he wants to present to a Verifier
and by doing so enhances his privacy. 
Yet, in the event he only does a partial disclosure of those claims, he is still leaking to the Verifier the number of claims he possesses ; which by definition diminishes his privacy.
Also, each of those redacted claims, even though it is not presented to a Verifier, will inflate the presented token (SD-KBT), 34 bytes at a time.

To tackle both those problems, we propose here, instead of individually hashing the claims and placing them in a CBOR array, to place them on a Merkle tree
and to only leave the root hash in the SD-CWT. Then, a presentation of this token will only need to present a Merkle proof of the claim(s) to attest for their integrity.
This proof is logarithmically smaller than having to enumerate all the redacted elements.

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
