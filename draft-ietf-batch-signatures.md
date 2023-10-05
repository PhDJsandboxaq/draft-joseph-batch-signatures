---
title: TurboTLS for faster connection establishment
abbrev: ietf-turbotls-design
docname: draft-ietf-turbotls-design-latest
date: 2023-09-05
category: info

? ipr: trust200902
keyword: Internet-Draft

? stand_alone: yes
? pi: [toc, sortrefs, symrefs]

submissionType: IETF

author:
  -
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
  CECPQ1:
    target: https://security.googleblog.com/2016/07/experimenting-with-post-quantum.html
    title: Experimenting with Post-Quantum Cryptography
    author:
      -
        ins: M. Braithwaite
    date: 2016-07-07
  CECPQ2:
    target: https://www.imperialviolet.org/2018/12/12/cecpq2.html
    title: CECPQ2
    author:
      -
        ins: A. Langley
    date: 2018-12-12
  DODIS: DOI.10.1007/978-3-540-30576-7_11
  DOWLING: DOI.10.1007/s00145-021-09384-1
  ETSI:
    target: https://www.etsi.org/images/files/ETSIWhitePapers/QuantumSafeWhitepaper.pdf
    title: "Quantum safe cryptography and security: An introduction, benefits, enablers and challengers"
    author:
      -
        role: editor
        ins: M. Campagna
      -
        ins: others
    seriesinfo: ETSI White Paper No. 8
    date: 2015-06
  EVEN: DOI.10.1007/978-1-4684-4730-9_4
  EXTERN-PSK: RFC8773
  FLUHRER:
    target: https://eprint.iacr.org/2016/085
    title: "Cryptanalysis of ring-LWE based key exchange with key share reuse"
    author:
      -
        ins: S. Fluhrer
    seriesinfo: Cryptology ePrint Archive, Report 2016/085
    date: 2016-01
  FO: DOI.10.1007/s00145-011-9114-1
  FRODO: DOI.10.1145/2976749.2978425
  GIACON: DOI.10.1007/978-3-319-76578-5_7
  HARNIK: DOI.10.1007/11426639_6
  HHK: DOI.10.1007/978-3-319-70500-2_12
  HPKE: RFC9180
  IKE-HYBRID: I-D.tjhai-ipsecme-hybrid-qske-ikev2
  IKE-PSK: RFC8784
  KIEFER: I-D.kiefer-tls-ecdhe-sidh
  Kyber:
    target: https://csrc.nist.gov/CSRC/media/Projects/post-quantum-cryptography/documents/round-3/submissions/Kyber-Round3.zip
    title: Crystals-Kyber NIST Round 3 submission
    author:
      -
        ins: Roberto Avanzi, Joppe Bos, Léo Ducas, Eike Kiltz, Tancrède Lepoint, Vadim Lyubashevsky, John M. Schanck, Peter Schwabe, Gregor Seiler, Damien Stehlé
    date: 2020-10-01
  LANGLEY:
    target: https://www.imperialviolet.org/2018/04/11/pqconftls.html
    title: Post-quantum confidentiality for TLS
    author:
      -
        ins: A. Langley
    date: 2018-04-11
  LUCKY13:
    target: https://ieeexplore.ieee.org/iel7/6547086/6547088/06547131.pdf
    title: "Lucky Thirteen: Breaking the TLS and DTLS record protocols"
    author:
    -
      ins: N. J. Al Fardan
    -
      ins: K. G. Paterson
  NIELSEN:
    title: Quantum Computation and Quantum Information
    author:
      -
        ins: M. A. Nielsen
      -
        ins: I. L. Chuang
    seriesinfo: Cambridge University Press
    date: 2000
  NIST:
    target: https://www.nist.gov/pqcrypto
    title: Post-Quantum Cryptography
    author:
      org: National Institute of Standards and Technology (NIST)
  NIST-FIPS-202:
    target: https://doi.org/10.6028/NIST.FIPS.202
    title: "SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions"
    author:
      org: National Institute of Standards and Technology (NIST)
    date: 2015-08
  NIST-SP-800-56C:
    target: https://doi.org/10.6028/NIST.SP.800-56Cr2
    title: Recommendation for Key-Derivation Methods in Key-Establishment Schemes
    author:
      org: National Institute of Standards and Technology (NIST)
    date: 2020-08
  NIST-SP-800-135:
    target: https://doi.org/10.6028/NIST.SP.800-135r1
    title: Recommendation for Existing Application-Specific Key Derivation Functions
    author:
      org: National Institute of Standards and Technology (NIST)
    date: 2011-12
  OQS-102:
    target: https://github.com/open-quantum-safe/openssl/tree/OQS-OpenSSL_1_0_2-stable
    title: OQS-OpenSSL-1-0-2_stable
    author:
      org: Open Quantum Safe Project
    date: 2018-11
  OQS-111:
    target: https://github.com/open-quantum-safe/openssl/tree/OQS-OpenSSL_1_1_1-stable
    title: OQS-OpenSSL-1-1-1_stable
    author:
      org: Open Quantum Safe Project
    date: 2022-01
  OQS-PROV:
    target: https://github.com/open-quantum-safe/oqs-provider/
    title: OQS Provider for OpenSSL 3
    author:
      org: Open Quantum Safe Project
    date: 2023-07
  PST: DOI.10.1007/978-3-030-44223-1_5
  RACCOON:
    target: https://raccoon-attack.com/
    title: "Raccoon Attack: Finding and Exploiting Most-Significant-Bit-Oracles in TLS-DH(E)"
    author:
    -
      ins: R. Merget
    -
      ins: M. Brinkmann
    -
      ins: N. Aviram
    -
      ins: J. Somorovsky
    -
      ins: J. Mittmann
    -
      ins: J. Schwenk
    date: 2020-09
  S2N:
    target: https://aws.amazon.com/blogs/security/post-quantum-tls-now-supported-in-aws-kms/
    title: Post-quantum TLS now supported in AWS KMS
    author:
      org: Amazon Web Services
    date: 2019-11-04
  SCHANCK: I-D.schanck-tls-additional-keyshare
  WHYTE12: I-D.whyte-qsh-tls12
  WHYTE13: I-D.whyte-qsh-tls13
  XMSS: RFC8391
  ZHANG: DOI.10.1007/978-3-540-24632-9_26

--- abstract



--- middle

# Introduction {#introduction}





## Terminology {#terminology}

UDP
TCP
TLS
QUIC
DNS
connection-based protocol
connectionless protocol


## Motivation for handshaking over UDP {#motivation}




## Scope {#scope}

This document focuses on TurboTLS {{TurboTLS}}. It covers everything needed to achieve the handshaking portion of a TLS connection over UDP, including

- **Construction in principle:** It provides an outline of which flows are sent over UDP, which are sent over TCP and in what order.
  



## Goals {#goals}

- **High performance:** Successful use of TurboTLS removes one round trip and should cut handshaking time by up to 50%. However in the worst case, when the fallback mechanism to TLS-over-TCP is used, there should be only a minimal impact on latency.




# Transport Layer Security {#TLS}


# Construction for TurboTLS {#construction}           



## TurboTLS support advertisment {#Construction-advertisment}


# Discussion {#discussion}


# Security Considerations {#security-considerations}

## Transparent proxying {#security-proxy}

## Denial-of-Service {#security-DoS}

# Acknowledgements



--- back

# Related work {#related-work}


