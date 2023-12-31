---
title: Batched Signatures
abbrev: joseph-tls-batch-signatures
docname: draft-joseph-tls-batch-signatures-latest
date: 2023-11-05
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

informative:
  AABBHHJM23:
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
    target: https://datatracker.ietf.org/doc/html/draft-ietf-tls-batch-signing-00#section-2
    title: "Batch Signing for TLS: draft-ietf-tls-batch-signing-00"
    date: 2023
    author:
      -
        ins: David Benjamin
  BEN23:
    target: https://datatracker.ietf.org/doc/draft-davidben-tls-merkle-tree-certs/
    title: "Merkle Tree Certificates for TLS"
    date: 2023-09-08
    author:
      -
        ins: David Benjamin
  CRYSTALS-DILITHIUM: DOI.10.46586/tches.v2018.i1.238-268
  FALCON: DOI.10.6028/nist.fips.206
  GK12: DOI.10.1007/s13389-012-0037-z
  NSM10:
    target: https://www.whitehouse.gov/briefing-room/statements-releases/2022/05/04/national-security-memorandum-on-promoting-united-states-leadership-in-quantum-computing-while-mitigating-risks-to-vulnerable-cryptographic-systems/
    title: "National Security Memorandum on Promoting United States Leadership in Quantum Computing While Mitigating Risks to Vulnerable Cryptographic Systems"
    date: 2010-05-04
    author:
      -
        ins: Shalanda D. Young
  SPHINCS: DOI.10.1145/3319535.3363229
  SUPERCOP:
    target: https://bench.cr.yp.to/supercop.html
    title: "SUPERCOP: System for unified performance evaluation related to cryptographic operations and primitives."
    date: 2018
    author:
      -
        ins: Daniel J. Bernstein
      -
        ins: Tanja Lange

--- abstract

This document proposes a construction for batch signatures where a single,
potentially expensive, "base" digital signature authenticates a Merkle tree
constructed from many messages.

Discussion of this work is encouraged to happen on the IETF TLSWG mailing list
tls@ietf.org or on the GitHub repository which contains the draft:
https://github.com/PhDJsandboxaq/draft-joseph-batch-signatures

--- middle

# Introduction {#introduction}

The computational complexity of unkeyed and symmetric cryptography is known
to be significantly lower than asymmetric cryptography. Indeed, hash functions,
stream or block ciphers typically require between a few cycles {{AES-NI}} to
a few hundred cycles {{GK12}}, whereas key establishment
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
Falcon {{FALCON}} and SPHINCS {{SPHINCS}} -- were
selected, and migration from current standards to these new algorithms is already
underway {{NSM10}}. One of the key issues when considering migrating to PQC is
that the computational costs of the new digital signature algorithms are significantly
higher than those of ECDSA; the fastest currently-deployed primitive for signing.
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

* Collision resistance - no two inputs _x1, x2_ should map to the same output hash
  (regardless of choice of key in the case of a keyed hash).
* Preimage resistance - it should be difficult to guess the input value _x_ for a hash
  function given only its output _H(x)_.
* Second preimage resistance - given an input _x1_, it should be difficult to find
  another distinct input _x2_ such that _H(x1)=H(x2)_.
* Target collision resistance - choose input _x1_. Then given a key _v_, find _x2_
 such that _Hv(x1) = Hv(x2)_.

Target collision resistance is a weaker requirement than collision resistance but is
sufficient for signing purposes. Tweakable hash functions enable us to tightly achieve
TCR even in multi-target settings where an adversary can attack one out of many targets.
Specifically, the property achieved in the following is _single-function multi-target
collision resistance_ (SM-TCR) which is described more formally in {{AABBHHJM23}}.

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
the security proofs the reader should see {{AABBHHJM23}}.

# Batch signature construction {#construction}

## Batch signature definition {#construction-batch-signature-definition}

We define a batch signature as a triple of algorithms
- **Batch signature** a batch signature scheme is comprised of _KeyGen_, _BSign_, _Verify_ whereby:
  - _KeyGen(k)_ outputs a keypair _sk, pk_ where _k_ is a security parameter
  - _BSign(sk, M)_ the batch signing algorithm takes as input a list of messages _M = {msg-i}_
    and outputs a list of signatures _S=\{sig-i\}_. We write _S <-- BSign(sk,M)_
  - PVerify(pk, sig, msg)_. We call the verification _PVerify_ to represent Path Verification,
    because in the construction outlined below, verification involves an extra step of verifying
    a sibling path, which _Verify_ in the ordinary DSA setting does not do.

## Merkle tree batch signature {#construction-merkle-tree}

Our construction relies on a Merkle tree. When addressing nodes in a Merkle tree of height _h_
with _N_ leaves, we may label nodes and leaves in the tree by their position: _T\[i,k\]_ is the
_i_-th node at height _k_, counting from left to right and from bottom upwards (i.e. leaves are
on height _0_ and the root is on height _h_. We illustrate this in {{fig-merkle-tree}}.

Let _Sig=(KeyGen, Sign, Verify)_ be a DSA as defined in {{Preliminaries-signatures}}, let _thash_
be a tweakable hash function as defined in {{Preliminaries-tweakable-hashes}}. We define our batch
signature scheme  _BSig = (KeyGen, BSign, BVerify_ with _KeyGen := Sig.KeyGen_ and _BSign, Verify_
as in {{construction-batch-signature-definition}} respectively.

Here we describe the case of binary Merkle trees. In the case where _N_ is not a power of _2_, one
can pad the tree by repeating leaves, or else continue with an incomplete tree.

## Signing {#construction-signing}

_BSign(sk, M=\[msg-0,...,msg-N-1\])_ where _N=2^n_. We first treat the case that _N_ is a power of
_2_, and then consider incomplete trees using standard methods.

### Tree computation {#construction-tree}

- **Initialize tree** _T[ ]_, which is indexed by the level, and then the row index, e.g. _T[3,5]_
  is the fifth node on level 3 of _T_. Height _h <-- log2(N)_
- **Tree identifier** Sample a tree identifier _id <--$ {0,1}^k_
- **Generate leaves** For leaf _i in [0,...,N-1]_, sample randomness _r-i <--$ {0,1}^k_. Then set
  _T[0,i] = H(id, 0, i, r-i, msg-i)._
- **Populate tree** For levels _l in [1,..., h]_ compute level _l_ from level _l-1_ as follows:
  - Initialize level _l_ with half as many elements as level _l-1_.
  - For node _j_ on level _l_, set _left=T[l-1, 2j]_ and _right=T[l-1, 2j+1]_
  - _id_ is the public parameter, _(1, l, j)_ is the tweak.
  - _T[l, j] <-- H(id, 1, l, j, left, right)_
- **Root** set _root <-- T[h,0]_

### Signature construction {#construction-signature}

- **Sign the root** Use the base DSA to sign the root of the Merkle tree _rootsig <-- Sign(sk, id, root, N)_
- **Sibling path** For each leaf: The sibling path is an array of _h-1_ hashes. Compute the sibling path as follows:
    * Initialize _path-i <-- \[\]_
    * For _l in \[0, ..., N-1\]_, set _j <--floor(i / 2^l)_. If _j mod 2 = 0_ then _path-i\[l\]=T\[l,j+1\]_, else _path-i\[l\]=T\[l,j-1\]_
- **Generate batch signatures** _bsig-i <-- (id, N, sig, i, r-i, path-i)_
- **Return batch of signatures** batch signatures are \{bsig-1, ..., bsig-N\}

{{fig-merkle-tree}} illustrates the construction of the Merkle tree and the signature of the root.

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

Verification proceeds by first reconstructing the root hash via the leaf information (public parameters, tweak, message)
and iteratively hashing with the nodes provided in the sibling path. Next the base verification algorithm
is called to verify the base DSA signature of the root.

- **Generate leaf hash** Get hash from public parameter, tweak, and message _h <-- H(id, 0, i, r, msg)_.
- **Reconstruct root** Set _l=0_. For _l in [ 1, ..., h]_ set _j <-- floor(i/2^l)_.
  - If _j mod 2 = 0_: set _h <-- H(id, 1, l, j, h, path[l])_.
  - If _j mod 2 = 1_: set _h <-- H(id, 1, l, j, path[l], h)_.
- **Verify root** Return _Verify(pk, sig, h)_.

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

Correctness can be broken down into correctness of the base DSA, and correctness of the Merkle tree.
Correctness is considered a security property, and is generally proven for each DSA, so we only need
to demonstrate correctness of the Merkle tree root. The Merkle proof assures that a message is a leaf
at a given index or address, in the tree identified by _id_, and nothing more.

Hash functions are symmetric and therefore deterministic in nature, therefore, given the correct
sibling path as part of the signer's signature, will generate the Merkle tree root correctly. Given
an accepting (message, signature) pair, so long as one uses a hash function that provides second
preimage resistance (from which the tweakable hash then provides SM-TCR), this guarantees that - except
with negligible probability - the proof must be valid only for the message provided.

## Domain separation {#dom-sep}

Our construction uses tweakable hashfunctions which take as input a public parameter _id_, a tweak _t_,
and a message _msg_. The public parameter domain separates between Merkle trees used in the same
protocol. In {{BEN20}} it is suggested that TLS context strings are used to prevent cross-protocol
attacks, however we note here that _msg_, which is the full protocol transcript up to that point,
includes such protocol context strings. Therefore domain separation is handled implicitly. However
in an idea world all protocols would agree on a uniform approach to domain separation, and so we
invite comment on how to concretely handle this aspect.

## Target collision resistance vs collision resistance {#tcr-vs-cr}

Instead of collision resistance, relying on the weaker assumption of TCR, and more specifically SM-TCR,
increases the attack complexity of finding a forgery and breaking the scheme. The result is that
smaller hash outputs are required to achieve an equivalent security level. This key modification
versus {{BEN20}} thus provides smaller signatures or higher security for the same sized signatures.

The reduction in signature size is _(h-1) * d_ where _h_ is the tree height and _d_ is the difference
between the hash output length based on SM-TCR and that based on collision resistance. Usually TCR
implies using, for example, 128b digests to achieve 128b security, whereas basing on CR would require
256b digests. This change therefore in essence halves the size of the Merkle tree proofs.

# Post-quantum and hybrid signatures {#pqc}

**Digital signatures** The transition to post-quantum cryptography (PQC) coincides with increasing demands
on performance and throughput. Post-quantum signatures suffer worse performance both on computation and
public key/signature sizes versus their quantum-vulnerable counterparts which are based on elliptic curve
cryptography and RSA. As a result, techniques to boost performance are especially relevant in this case.
In {{pqc-signature-sizes}} one can see that the extra size cost due to the sibling path is roughly
independent of the algorithm used (therefore relatively much smaller for PQC).

**Hash functions** In contrast to DSAs, Hash functions are purely symmetric cryptography which is weakened
but not broken by quantum computers. Therefore the Merkle tree construction is not affected in the
post-quantum era, other than a doubling of parameters (in the worst case) to achieve the same security level.

## Signature sizes {#pqc-signature-sizes}

In {{Table1}} one can see the size of a batch signature which signs 16 or 32 transcripts, relative to the
size of the underlying primitive. For post-quantum schemes the overhead in size is relatively small due
to the much larger sizes of the base DSAs.

~~~
+--------------------+-----+------+-------+----+--------+
|       Scheme       |  k  | |vk| | |sig| | N  | |bsig| |
+--------------------+-----+------+-------+----+--------+
| ECDSA P256         | 128 |   64 |    64 | 32 |    180 |
| Dilithium2         | 128 | 1312 |  2420 | 32 |   2536 |
| Dilithium5         | 256 | 2592 |  4595 | 32 |   4823 |
| Falcon-512         | 128 |  897 |   666 | 16 |    766 |
| Falcon-1024        | 256 | 1793 |  1280 | 32 |   1508 |
| Falcon-512-fpuemu  | 128 |  897 |   666 | 16 |    766 |
| Falcon-1024-fpuemu | 256 | 1793 |  1280 | 16 |   1476 |
| SPHINGS+-128f      | 128 |   32 | 17088 | 16 |  17188 |
| SPHINCS-256f       | 256 |   64 | 49856 | 32 |  50084 |
+--------------------+-----+------+-------+----+--------+
~~~
{: #Table1 title="Batch signature sizes "}

## Hybrid signatures {pqc-hybrid}

A likely mode of transition to PQC will be via 'hybrid mode', where data is protected independently via two
algorithms, one quantum-vulnerable (but well studied and understood) and one PQC algorithm. This is to
mitigate the risk of a complete break - classical or quantum - of a PQC algorithm. Breaking a hybrid scheme
should imply breaking _both_ of the algorithms used.

We do not discuss the details of such hybrid signatures or hybrid certificates in this document, but simply
state that so long as the hybrid scheme adheres to the API described above, the Batch signature Merkle tree
construction described in this document remains unaltered. Explicitly, the root is generated via the procedure
of {{construction-tree}}. Then the root is signed by the hybrid DSA, whose functions _KeyGen_, _Sign_, _Verify_
are constructed via some composition of _KeyGen_, _Sign_, _Verify_ for a PQC algorithm and _KeyGen_, _Sign_,
_Verify_ for some presently-used algorithm.

## Privacy {#privacy}

In {{AABBHHJM23}} two privacy notions are defined:

- **Batch Privacy** can one cannot deduce whether two messages were signed in the same batch.
- **weak Batch Privacy** for two messages signed in the same batch, if one is given the signature for one
-  message, it does not leak any information about the other message, for which no signature is available.

The authors prove in {{AABBHHJM23}} that this construction achieves the weaker variant, but not full Batch
Privacy.

# Relationship to Merkle Tree Certificates {#relationship-MTC}

A Merkle tree construction for TLS certificates {{BEN23}} is being developed at the time of writing, by the
same author of the original Merkle tree signing draft {{BEN20}}. The construction bears strong similarities
to the current proposal. In ordinary TLS certificates, a Certificate Authority (CA) signs a certificate which
asserts that a public key belongs to a given subscriber. In the Merkle tree construction, many certificates
are batched together using a similar Merkle tree construction to the one presented in this document. The CA
then signs only the root of the Merkle tree, and returns (root signature + sibling path) to the subscriber.

A client verifies a server's identity by:

- Verifying a server's signature: the server signs the TLS transcript up to that point with their private
  key and the client verifies with the server's public key _pk_.
- Verifying that the public key belongs to the server by verifying the trusted CA's signatures certificate
  which states that the server owns _pk_.
  - Doing this repeatedly in the case of certificate chains until reaching a root CA.

The document of {{BEN23}} relates specifically to signing certificates, the second bullet above, whereas the
constructions of {{BEN20}} and this document pertain to a server authenticating itself online, relating to
the first bullet above. The two have slightly different usecases, which both benefit from Merkle tree
constructions under different scenarios.

Cases where Merkle tree certificates may be appropriate have certain properties:

- Certificates are short-lived.
- Certificates are issued after a significant delay, e.g. around one hour.
- Batch sizes can be estimated to be up to 2^24 (based on unexpired number of certificates in certificate
  transparency logs)

Cases where TLS batch signing may be appropriate differ slightly, for example:

- High throughput servers and load balancers - in particular when rate of incoming signing requests exceeds
  _(time * threads)_ where _time_ is the average time for a signing thread to generate a signature, and
  _threads_ is the number of available signing threads.
- In scenarios where the latency is not extremely sensitive: waiting for signatures to arrive before constructing
  a Merkle tree incurs a small extra latency cost which is amortised by the significant extra throughput achievable.
- Batch sizes are likely to be smaller than the usecase for Merkle tree certificates. Batch sizes of 16 or 32 can
  already improve throughput by an order of magnitude.

# Acknowledgements {#acknowledgements}


--- back

# Related work {#related-work}


