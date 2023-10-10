---
title: TurboTLS for faster connection establishment
abbrev: ietf-turbotls-design
docname: draft-ietf-turbotls-design-latest
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
    ins: C. Aguilar-Melchor
    name: Carlos Aguilar-Melchor
    organization: SandboxAQ
    email: carlos.aguilar@sandboxaq.com
  -

normative:
  TLS13: RFC8446

informative:
  AVIRAM:
    target: https://mailarchive.ietf.org/arch/msg/tls/F4SVeL2xbGPaPB2GW_GkBbD_a5M/
    title: "[TLS] Combining Secrets in Hybrid Key Exchange in TLS 1.3"
    date: 2021-09-01
    author:
      -
        ins: Nimrod Aviram
      -
        ins: Benjamin Dowling
      -
        ins: Ilan Komargodski
      -
        ins: Kenny Paterson
      -
        ins: Eyal Ronen
      -
        ins: Eylon Yogev
  BCNS15: DOI.10.1109/SP.2015.40
  BERNSTEIN: DOI.10.1007/978-3-540-88702-7
  BINDEL: DOI.10.1007/978-3-030-25510-7_12
  CAMPAGNA: I-D.campagna-tls-bike-sike-hybrid
  
--- abstract
This document proposes a construction for batch signatures where a single, potentially expensive, "inner" digital signature authenticates a Merkle tree constructed from many messages.

--- middle

# Introduction {#introduction}

The computational complexity of unkeyed and symmetric cryptography is known to be significantly lower than asymmetric cryptography. Indeed, hash functions, stream or block ciphers typically require between a few cycles~\cite{aes-ni} to a few hundred cycles~\cite{gueron2012parallelizing}, whereas key establishment and digital signature primitives require between tens of thousands to hundreds of millions of cycles~\cite{djb2018supercop}. In situations where a substantial volume of signatures must be handled -- e.g.~a Hardware Security Module (HSM) renewing a large set of short-lived certificates or a load balancer terminating a large number of TLS connections per second -- this may pose serious limitations on scaling these and related scenarios.

These challenges are amplified by upcoming public-key cryptography standards: In July 2022, the US National Institute of Standards and Technology (NIST) announced four algorithms for post-quantum cryptography (PQC) standardisation. In particular, three digital signature algorithms -- Dilithium~\cite{NISTPQC:CRYSTALS-DILITHIUM22}, Falcon~\cite{NISTPQC:FALCON22} and SPHINCS$^+$~\cite{NISTPQC:SPHINCS+22} -- were selected, and migration from current standards to these new algorithms is already underway~\cite{nsm_10}. One of the key issues when considering migrating to PQC is that the computational costs of the new digital signature algorithms are significantly higher than those of ECDSA\@; the fastest currently-deployed primitive for signing. This severely impacts the ability of systems to scale and inhibits their migration to PQC, especially in higher-throughput settings.



## Conventions and definitions {#Conventions}

### Signatures {#Conventions-signatures}

- **DSA** Digital Signature Algorithm, a public key cryptography primitive consisting of a triple of algorithms _KeyGen_, _Sign_, _Verify_ whereby:
  - $KeyGen(k)$ outputs a keypair _sk, pk_ where _k_ is a security parameter
  - $Sign(msg, sk) = s$
  - $Verify(s, msg, pk) = b$. When $b=1$, the result is ACCEPT which occurs on receipt of message, correctly signed with the secret key $sk$ corresponding to $pk$. Otherwise the result is REJECT when $b=0$.

 ### Hash functions {#Conventions-hashes}
 
 In this work we consider _tweakable_ hash functions. These are keyed hash functions that take an additional input which can be thought of as a domain separator (while the key or public parameter serves as a separator between users). Tweakable hash functions allow us to tightly achieve target collision resistance even in multi-target settings where an adversary wins when they manage to attack one out of many targets.

 **Tweakable Hash functions** A _tweakable hash function_ is a tuple of algorithms $H=(KeyGen, Eval)$ such that:
 - $KeyGen$ takes the security parameter $k$ and outputs a (possibly empty) public parameter $p$. We write $p <-- KeyGen(k)$.
 - $Eval$ is deterministic and takes public parameters $p$, a tweak $t$, an input $msg$ in $[0,1]^m$ and returns a hash value $h$. We write $h <-- Eval(p,t,msg)$ or simply $h <-- H(p,t,msg)$.

 **Properties of hash functions**

## Motivation for batching messages before signing {#motivation}

Batch signing enables the scaling of slow algorithms by signing large batches of messages simultaneously. This comes at a relatively low latency cost, and significant throughput improvements. These advantages are particularly pertinent when considering the use of post-quantum signatures, such as those being standardised by NIST at the time of writing.


## Scope {#scope}

This document describes a construction for Batch signatures based upon a Merkle tree design. The total signature consists of a signature of the root of the Merkle tree, as well as the sibling path of the individual message. We discuss the applicability of such signatures to various protocols, but only at a high level. The document describes a scheme which enables smaller signatures than outlined in {{Ben20}} by relying not on hash collision resistance, but instead on target collision resistance, however for the security proofs the reader should see \cite{original paper!}.

# Batch signature construction {#construction}           



## Sign {#Construction-sign}

## Verify {#Construction-verify}


# Discussion {#discussion}

<!-- Hybrid?  Can just do any other hybrid construction scheme, have BSign just call that internally as S.Sign, and S.Verify. We should consider the separability concerns etc though. -->
<!-- How are tree id's generated in a cross-instantiation-secure way? Are we worried about collisions? λ only ranges up to 256. -->
<!-- Maybe domain separate the tree hash function H with a label prefix before the rest -->
<!-- How does this differ from Merkle Tree Certificates? 'It has a merkle tree in it' -->



# Security Considerations {#security-considerations}

## Correctness

## Domain separation

## Target collision resistance vs collision resistance

## Privacy

# Acknowledgements



--- back

# Related work {#related-work}


