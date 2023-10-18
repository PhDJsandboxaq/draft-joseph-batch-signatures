---
title: Batched Signatures
abbrev: ietf-batched-signatures
docname: draft-ietf-batch-signatures-latest
date: 2023-09-05
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

submissionType: IETF

author:
  -
    ins: D. Joseph
    name: David Joseph
    organization: SandboxAQ
    email: dj@sandboxaq.com
  -
    ins: D. Connolly
    name: Deirdre Connolly
    organization: SandboxAQ
    email: deirdre.connolly@sandboxquantum.com
  -
    ins: C. Aguilar-Melchor
    name: Carlos Aguilar-Melchor
    organization: SandboxAQ
    email: carlos.aguilar@sandboxaq.com

normative:
  TLS13: RFC8446

informative:
  AES-NI:
    target: https://www.intel.cn/content/dam/develop/external/us/en/documents/10tb24-breakthrough-aes-performance-with-intel-aes-new-instructions-final-secure-165940.pdf
    title: "Breakthrough AES Performance with Intel AES New Instructions"
    date: 2010
    author:
      -
        ins: Kahraman Akdemir
      -
        ins: Martin Dixon
      -
        ins: Wajdi Feghali
      -
        ins: Patrick Fay
      -
        ins: Vinodh Gopal
      -
        ins: Jim Guilford
      -
        ins: Erdincand Ozturk
      -
        ins: Gil Wolrich
      -
        ins: Ronen Zohar
  BEN20:
    target: TODO
    title: TODO
    date: 2023
    author:
      -
        ins: DAVID FIX THIS REFERENCE
  GUE2012PARALLEL: DOI.10.1007/s13389-012-0037-z
  SUPERCOP:
    target: https://bench.cr.yp.to/supercop.html
    title: "SUPERCOP: System for unified performance evaluation related to cryptographic operations and primitives."
    date: 2018
    author:
      -
        ins: Daniel J. Bernstein
      -
        ins: Tanja Lange
  CRYSTALS-DILITHIUM: DOI.10.46586/tches.v2018.i1.238-268
  FALCON: DOI.10.6028/nist.fips.206
  SPHINCS+: DOI.10.1145/3319535.3363229
  NSM10:
    target: https://www.whitehouse.gov/briefing-room/statements-releases/2022/05/04/national-security-memorandum-on-promoting-united-states-leadership-in-quantum-computing-while-mitigating-risks-to-vulnerable-cryptographic-systems/
    title: "National Security Memorandum on Promoting United States Leadership in Quantum Computing While Mitigating Risks to Vulnerable Cryptographic Systems"
    date: 2010-05-04
    author:
      -
        ins: Shalanda D. Young
  BATCHSIGREV:
    target: https://eprint.iacr.org/2023/492
    title: "Batch Signatures, Revisited"
    date: 2023
    author:
      -
        ins: Carlos Aguilar-Melchor
      -
        ins: Martin Albrecht
      -
        ins: Thomas Bailleux
      -
        ins: Nina Bindel
      -
        ins: James Howe
      -
        ins: Andreas Huelsing
      -
        ins: David Joseph
      -
        ins: Marc Manzano

--- abstract

This document proposes a construction for batch signatures where a single,
potentially expensive, "inner" digital signature authenticates a Merkle tree
constructed from many messages.

--- middle

# Introduction {#introduction}

The computational complexity of unkeyed and symmetric cryptography is known
to be significantly lower than asymmetric cryptography. Indeed, hash functions,
stream or block ciphers typically require between a few cycles {{AES-NI}} to
a few hundred cycles {{GUE2012PARALLEL}}, whereas key establishment
and digital signature primitives require between tens of thousands to hundreds
of millions of cycles {{SUPERCOP}}. In situations where a substantial
volume of signatures must be handled -- e.g. a Hardware Security Module (HSM)
renewing a large set of short-lived certificates or a load balancer terminating
a large number of TLS connections per second -- this may pose serious limitations
on scaling these and related scenarios.

These challenges are amplified by upcoming public-key cryptography standards: in
July 2022, the US National Institute of Standards and Technology (NIST) announced
four algorithms for post-quantum cryptography (PQC) standardisation. In particular,
three digital signature algorithms -- Dilithium {{CRYSTALS-DILITHIUM}},
Falcon {{FALCON}} and SPHINCS+ {{SPHINCS+}} -- were
selected, and migration from current standards to these new algorithms is already
underway {{NSM10}}. One of the key issues when considering migrating to PQC is
that the computational costs of the new digital signature algorithms are significantly
higher than those of ECDSA\@; the fastest currently-deployed primitive for signing.
This severely impacts the ability of systems to scale and inhibits their migration
to PQC, especially in higher-throughput settings.


# Preliminaries {#Preliminaries}

## Signatures {#Preliminaries-signatures}

- **DSA** Digital Signature Algorithm, a public key cryptography primitive
  consisting of a triple of algorithms _KeyGen_, _Sign_, _Verify_ whereby:
  - _KeyGen(k)_ outputs a keypair _sk, pk_ where _k_ is a security parameter
  - _Sign(sk, msg) = s_
  - _Verify(pk, s, msg) = b_. When _b=1_, the result is ACCEPT which occurs
    on receipt of message, correctly signed with the secret key _sk_ corresponding
    to _pk_. Otherwise the result is REJECT when _b=0_.

## Hash functions {#Preliminaries-hashes}

In this work we consider _tweakable_ hash functions. These are keyed hash functions
that take an additional input which can be thought of as a domain separator (while
the key or public parameter serves as a separator between users). Tweakable hash
functions allow us to tightly achieve target collision resistance even in multi-target
settings where an adversary wins when they manage to attack one out of many targets.

- **Keyed Hash function** A keyed hash function is one that outputs a hash that depends
both on a message _msg_ and a key _v_ that is shared by both the hash generator and the
hash validator. It is easiest to compute via prepending the key to the message before
computing the hash _h <-- H(v | msg)_. Let _Hv_ denote the family of hash functions
keyed with _v_.

### Hash function properties {#Preliminaries-hash-properties}

* Collision resistance - no two inputs _x1, x2_ should map to the same output hash (regardless of choice of key in the case of a keyed hash).
* Preimage resistance - it should be difficult to guess the input value _x_ for a hash function given only its output _H(x)_.
* Second preimage resistance - given an input _x1_, it should be difficult to find another distinct input _x2_ such that _H(x1)=H(x2)_.
* Target collision resistance - choose input _x1_. Then given a key _v_, find _x2_ such that _Hv(x1) = Hv(x2)_.

Target collision resistance is a weaker requirement than collision resistance but is sufficient for signing purposes. Tweakable hash functions enable us to tightly achieve TCR even in multi-target settings where an adversary can attack one out of many targets. Specifically, the property achieved in the following is _single-function multi-target collision resistance_ (SM-TCR) which is described more formally in {{BATCHSIGREV}}.

### Tweakable Hash functions {#Preliminaries-tweakable-hashes}

One form of keyed hash function is a tweakable hash function, which enables one to obtain SM-TCR:

 - **Tweakable Hash functions** A _tweakable hash function_ is a tuple of algorithms _H=(KeyGen, Eval)_ such that:
     - _KeyGen_ takes the security parameter _k_ and outputs a (possibly empty) public
   parameter _p_. We write _p <-- KeyGen(k)_.
     - _Eval_ is deterministic and takes public parameters _p_, a tweak _t_, an input _msg_
   in _\[0,1\]^m_ and returns a hash value _h_. We write _h <-- Eval(p,t,msg)_ or
   simply _h <-- H(p,t,msg)_.


## Motivation for batching messages before signing {#motivation}

Batch signing enables the scaling of slow algorithms by signing large batches of
messages simultaneously. This comes at a relatively low latency cost, and significant
throughput improvements. These advantages are particularly pertinent when considering
the use of post-quantum signatures, such as those being standardised by NIST at
the time of writing. In some applications, the amount of data that needs to be sent
can be reduced in addition; namely, if a given entity (is aware that it) receives
multiple signatures from the same batch. In this case, sending the signed root
multiple times is redundant and we can asymptotically reduce the amount of
received information to a few hashes per message.

## Scope {#scope}

This document describes a construction for batch signatures based upon a Merkle tree
design. The total signature consists of a signature of the root of the Merkle tree,
as well as the sibling path of the individual message. We discuss the applicability
of such signatures to various protocols, but only at a high level. The document describes
a scheme which enables smaller signatures than outlined in {{BEN20}} by relying not
on hash collision resistance, but instead on target collision resistance, however for
the security proofs the reader should see {{BATCHSIGREV}}.

# Batch signature construction {#construction}

## Batch signature definition {#construction-batch-signature-definition}

We define a batch signature as a triple of algorithms
- **Batch signature** a batch signature scheme is comprised of _KeyGen_, _BSign_, _Verify_ whereby:
  - _KeyGen(k)_ outputs a keypair _sk, pk_ where _k_ is a security parameter
  - _BSign(sk, M)_ the batch signing algorithm takes as input a list of messages _M = {msg-i}_ and outputs a list of signatures _S=\{sig-i\}_. We write _S <-- BSign(sk,M)_
  - PVerify(pk, sig, msg)_. We call the verification _PVerify_ to represent Path Verification, because in the construction outlined below, verification involves an extra step of verifying a sibling path, which _Verify_ in the ordinary DSA setting does not do.

## Merkle tree batch signature {#construction-merkle-tree}

Our construction relies on a Merkle tree. When addressing nodes in a Merkle tree of height _h_ with _N_ leaves, we may label nodes and leaves in the tree by their position: _T\[i,k\]_ is the _i_-th node at height _k_, counting from left to right and from bottom upwards (i.e. leaves are on height _0_ and the root is on height _h_. We illustrate this in {{fig-merkle-tree}}.

Let _Sig=(KeyGen, Sign, Verify)_ be a DSA as defined in {{Preliminaries-signatures}}, let _thash_ be a tweakable hash function as defined in {{Preliminaries-tweakable-hashes}}. We define our batch signature scheme  _BSig = (KeyGen, BSign, BVerify_ with _KeyGen := Sig.KeyGen_ and _BSign, Verify_ as in {{construction-batch-signature-definition}} respectively.

Here we describe the case of binary Merkle trees. In the case where _N_ is not a power of _2_, one can pad the tree by repeating leaves, or else continue with an incomplete tree.

## Signing {#construction-signing}

_BSign(sk, M=\[msg-0,...,msg-N-1\])_ where _N=2^n_. We first treat the case that _N_ is a power of _2_, and then consider incomplete trees using standard methods.

### Tree computation {#construction-tree}

- **Backwards compatibility:** Clients and servers who are "hybrid-aware", i.e., compliant with whatever hybrid key exchange standard is developed for TLS, should remain compatible with endpoints and middle-boxes that are not hybrid-aware.  The three scenarios to consider are:
    1. Hybrid-aware client, hybrid-aware server: These parties should establish a hybrid shared secret.
<!-- TODO there is an error with point 2 and most likely its subpoints. -->
```
- **Initialize tree** _T[]_, which is indexed by the level, and then the row index, e.g. _T[3,5]_ is the fifth node on level _3_ of _T_. Height _h <-- log2(N)_
- **Tree identifier** Sample a tree identifier _id <--$ {0,1}^k_
- **Generate leaves** For leaf _i in [0,...,N-1]_, sample randomness _r-i <--$ {0,1}^k_. Then set _T[0,i]=H(id | 0 | i | r-i | msg-i)_
- **Populate tree** For levels _l in [1,..., h]_ compute level _l_ from level _l-1_ as follows:
    1. Initialize level _l_ with half as many elements as level _l-1_.
    2. For node _j_ on level _l_, set _left=T[l-1, 2j]_ and _right=T[l-1, 2j+1]_
    3. _id_ is the public parameter, _(1, l, j)_ is the tweak.
    4. _T[l, j] <-- H(id | 1 | l | j | left | right)_
- **Root** set _root <-- T[h,0]_
```

### Signature construction {#construction-signature}

<!-- TODO there is an error with point 2 and most likely its subpoints. -->
```
- **Sign the root** Use the base DSA to sign the root of the Merkle tree _rootsig <-- Sign(sk, id, root, N)_
- **Sibling path** For each leaf: The sibling path is an array of _h-1_ hashes. Compute the sibling path as follows:
    * Initialize _path-i <-- \[\]_
    * For _l in \[0, ..., N-1\]_, set _j <--floor(i / 2^l)_. If _j mod 2 = 0_ then _path-i\[l\]=T\[l,j+1\]_, else _path-i\[l\]=T\[l,j-1\]_
- **Generate batch signatures** _bsig-i <-- (id, N, sig, i, r-i, path-i)_
- **Return batch of signatures** batch signatures are \{bsig-1, ..., bsig-N\}
```

~~~

  lvl3=root         (T30)--DSA--> sig
                     / \
  lvl2        T20----   ----T21
              /\             /\
             /  \           /  \
  lvl2    T10    T11     T12    T13
          /\     /\      /\      /\
         /  \   /  \    /  \    /  \
  lvl1 T00 T01 T02 T03 T04 T05 T06 T07
       |    |   |   |   |   |   |   |
       |            |               |
     H(_) ... H(id,0,3,r3,msg3)  ...H(_)

~~~
{: #fig-merkle-tree title="Merkle tree batch signature construction"}

## Verification {#construction-verification}

Verification proceeds by first reconstructing the root hash via the leaf information (public parameters, tweak, message) and iteravely hashing with the nodes provided in the sibling path. Next the base verification algorithm is called to verify the base DSA signature of the root.

<!-- TODO there is an error with point 2 and most likely its subpoints. -->
```
1. **Generate leaf hash** Get hash from public parameter, tweak, and message _h <-- H(id | 0 | i | r | msg)_.
2. **Reconstruct root** Set _l=0_. For _l in \[ 1,...,h\]_ set _j <-- floor(i / 2^l)_.
* If _j mod 2 = 0_: set _h <-- H(id | 1 | l | j | h | path\[l\])_
* If _j mod 2 = 1_: set _h <-- H(id | 1 | l | j | path\[l\] | h)_
3. **Verify root** Return _Verify(pk, sig, h)
```
# Discussion {#discussion}

<!-- Hybrid?  Can just do any other hybrid construction scheme, have BSign just call that internally as S.Sign, and S.Verify. We should consider the separability concerns etc though. -->
<!-- How are tree id's generated in a cross-instantiation-secure way? Are we worried about collisions? λ only ranges up to 256. -->
<!-- Maybe domain separate the tree hash function H with a label prefix before the rest -->
<!-- How does this differ from Merkle Tree Certificates? 'It has a merkle tree in it' -->



# Security Considerations {#security-considerations}

A reduction in certificate sizes is proposed by a new type of
CA (certificate authority) which would exclusively sign a new type
of batch oriented certificates.  Such a CA would only be used together
with a Certificate Transparency log, which changes the usual required
flows for authentication. The benefits obtained in certificate size and
verification/signature costs are significant. It also implies that the
main criterion for being on a same batch are: being signed by the same CA,
and being signed roughly at the same time.

## Correctness {#correctness}

## Domain separation {#dom-sep}

## Target collision resistance vs collision resistance {#tcr-vs-cr}

## Privacy {#privacy}

# Acknowledgements {#acknowledgements}


--- back

# Related work {#related-work}


