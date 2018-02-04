---
title: Verifiable Oblivious Pseudorandom Functions (VOPRFs)
abbrev: VOPRFs
docname: draft-wood-cfrg-voprf
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Goldberg
    name: Sharon Goldberg
    org: Boston University
    street: 111 Cummington St, MCS135
    city: Boston
    country: United States of America
    email: goldbe@cs.bu.edu
 -
    ins: N. Sullivan
    name: Nick Sullivan
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: nick@cloudflare.com
 -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, Califoarnia 95014
    country: United States of America
    email: cawood@apple.com

normative:
  RFC2119:
  RFC7748:
  RFC8032:
  PrivacyPass:
    title: Privacy Pass
    target: https://github.com/privacypass/challenge-bypass-server
  ChaumPedersen:
    title: Wallet Databases with Observers
    target: https://chaum.com/publications/Wallet_Databases.pdf
    authors:
      -
        ins: D. Chaum
        org: CWI, The Netherlands
      -
        ins: T. P. Pedersen
        org: Aarhus University, Denmark
  ChaumBlindSignature:
    title: Blind Signatures for Untraceable Payments
    target: http://sceweb.sce.uhcl.edu/yang/teaching/csci5234WebSecurityFall2011/Chaum-blind-signatures.PDF
    authors:
      -
        ins: D. Chaum
        org: University of California, Santa Barabara, USA

--- abstract

TODO

--- middle

# Introduction

A pseudorandom function (PRF) F(k, x) is an efficiently computable function with
secret key k on input x whose output is indistinguishable from any element in
F's range for some random key k. An oblivious PRF (OPRF) is a two-party protocol between 
a requester R and signer S wherein both parties cooperate to compute F(k, x) with S's 
secret key k and R's input x such only R learns F(k, x) without learning anything about k. 
Specifically, S uses its private key to help R compute F(k, x) output without learning 
the requestor's input. R blinds (and unblinds) its input such that S learns nothing.

Verifiable OPRFs (VOPRFs) are OPRF protocols wherein R can prove to S that its value 
F(k, x) was indeed computed over some input x without tracing back to the original computation. 
VOPRFs are useful for producing tokens that are verifiable by the signer yet 
unlinkable to the original requestor. They are used in the Privacy Pass 
protocol {{PrivacyPass}}. This document is structured as follows:

- Section {{background}}: Describe background, related related, and use cases of VOPRF protocols.
- Section {{properties}}: Discuss the security properties of such protocols.
- Section {{protocol}}: Specify a VOPRF protocol based on elliptic curve groups. 
<!-- - Section XXX: Specify a VOPRF extension that permits signers to prove the private key was integrated into the VOPRF computation. -->

## Terminology

The following terms are used throughout this document.

- PRF: Pseudorandom Function.
- OPRF: Oblivious PRF.
- VOPRF: Verifiable Oblivious Pseudorandom Function.
- Requestor (R): Protocol initiator when computing F(k, x).
- Signer (S): Holder of private VOPRF key k.
- NIZK: Non-interactive zero knowledge.
- DLEQ: Discrete Logarithm Equality.

## Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}.

# Background {#background}

VOPRFs are functionally related to RSA-based blind signature schemes, e.g., {{ChaumBlindSignature}}.
Such a scheme works as follows. 
Let m be a message to be signed by a server. It is assumed to be a member of the 
RSA group. Also, let N be the RSA modulus, and e and d be the public and private keys, 
respectively. The Requestor and Signer engage in the following protocol given input m. 

1. Requestor generates a random blinding element r from the RSA group, and 
compute m' = m^r (mod N). Send m' to the Signer.
2. Signer uses m' to compute s' = (m')^d (mod N), and sends s' to the Requestor.
3. Requestor removes the blinding factor r to obtain the original 
signature as s = (s')^(r^-1) (mod N).

By the properties of RSA, s is clearly a valid signature for m. 
(V)OPRF protocols differ from blind signatures in the same way that 
traditional digital signatures differ from PRFs. This is discussed more
in the following section.
  
# Security Properties {#properties}

The security properties of a VOPRF protocol with functionality y = F(k, x) are similar 
to that of a PRF. Specifically:

- Given value x, it is infeasible to learn y = F(k, x) without knowledge of k.
- Output y = F(k, x) is indistinguishable from a random value in the domain of F. 

Additionally, since this is an oblivious protocol, the following security properties
are required:

- S must learn nothing about the R's input.
- R must learn nothing about the S's private key.

# Elliptic Curve VOPRF Protocol {#protocol}

In this section we describe the ECVOPRF protocol. 
Let G be an elliptic curve group over base field F, of order p, 
with two distinct hash functions H_1 and H_2, where H_1 maps 
arbitrary input onto G and H_2 maps arbitrary input to a fixed-length output, e.g., SHA256.
Let L be the security parameter. Let k be the signer's secret key,
and Y = kG be its corresponding public key. Let x be the requestor's input to
the VOPRF protocol. (Commonly, it is a random L-bit string, though this 
is not required.) ECVOPRF begins with the requestor randomly blinding
its input for the signer. The latter then applies its secret key to the blinded
value and returns the result. To finish the computation, the requestor then 
removes its blind and hashes the result using H_2 to yield an output. 
This flow is illustrated below.

~~~
    Requestor               Signer
     r <-$ G
     M = rH_1(x) 
                   M
                ------->    
                           Z = kM
                   Z
                <-------
    N = Zr^(-1)    
~~~

The actual PRF function computed is as follows:

~~~
F(k, x) = y = H_2(k, kH_1(x))
~~~

Note that R finishes this computation upon receiving kH_1(x) from S. The output
from S is not the PRF value.

This protocol may be decomposed into a series of steps, as described below:

- ECVOPRF_Blind(x): Compute and return a blind, r, and blinded representation of x, M.
- ECVOPRF_Sign(M): Sign input M using secret key k to produce Z.
- ECVOPRF_Unblind(Z, r): Unblind blinded signature Z with blind r, yielding N.
- ECVOPRF_Finalize(N): Finalize N to produce PRF output F(k, x).

Protocol correctness requires that, for any key k, input x, and (r, M) = Blind(x), 
it must be true that:

~~~
Finalize(Sign(M), r) = F(k, x)
~~~

with overwhelming probability. 

## Algorithmic Details 

This section provides algorithms for each step in the VOPRF protocol.

1. Requestor computes T = H_1(t) and a random element r from G. (The latter is the
blinding factor.) The requestor computes M = rT.
2. Requestor sends M to the signer. 
3. Signer computes Z = xM = rxT. 
4. Signer sends (Z, Y) to the requestor.
5. Requestor unblinds Z to compute N = r^(-1)Z = xT.
6. Requestor outputs the pair (t, H_2(N)).

### ECVOPRF_Blind

~~~
Input:

 x - R's PRF input.

Output:

 r - Random scalar in [1, p - 1]
 M - Blinded representation of x using blind r, a point on G.

Steps:

 1.  r <-$ Fp
 2.  M := rx
 5.  Output (r, M)
~~~

### ECVOPRF_Sign

~~~
Input:

 M - Point on G.

Output:

 Z - Scalar multiplication of k and M, point on G.

Steps:

 1. Z := kM
 2. Output Z
~~~

### ECVOPRF_Unblind

~~~
Input:

 Z - Point on G.
 r - Random scalar in [1, p - 1].

Output:

 N - Unblinded signature, point on G.

Steps:

 1. N := (-r)Z
 2. Output N
~~~

### ECVOPRF_Finalize

~~~
Input:

 N - Point on G.

Output:

 y - Random element in {0,1}^L

Steps:

 1. y := H_2(N)
 2. Output y
~~~

## Group and Hash Function Instantiations

This section specifies supported VOPRF group and hash function instantiations.

EC-VOPRF-P256-SHA256:

- G: P-256
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA256

EC-VOPRF-P256-SHA512:

- G: P-256
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA512

EC-VOPRF-P384-SHA256:

- G: P-384
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA256

EC-VOPRF-P384-SHA512:

- G: P-384
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA512

EC-VOPRF-CURVE25519-SHA256:

- G: Curve25519 {{RFC7748}}
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA256

EC-VOPRF-CURVE25519-SHA512:

- G: Curve25519 {{RFC7748}}
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA512

EC-VOPRF-CURVE448-SHA256:

- G: Curve448 {{RFC7748}} 
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA256

EC-VOPRF-CURVE448-SHA512:

- G: Curve448 {{RFC7748}} 
- H_1: ((TODO: reference hash-to-curve draft))
- H_2: SHA512

# IANA Considerations

TODO

# Security Considerations

TODO

# Acknowledgments

TODO

--- back

# Discrete Logarithm Proofs

In some cases, it may be desirable for the Requestor to have proof that the Signer
used its private key to compute Z. Specifically, this is done by confirming
that log_G(Y) == log_G(Z). This may be used, for example, to ensure that the
Signer uses the same private key for computing the VOPRF output. This proof must
not reveal the Signer's long-term private key to the Requestor. Consequently,
we extend the protocol in the previous section with a (non-interactive) discrete 
logarithm equality (DLEQ) algorithm built on a Chaum-Pedersen {{ChaumPedersen}} proof.
This proof works as follows.

Input: 

  D: generator of G with prime order q
  E: orthogonal generator of G
  Y: Signer public key
  Z: Point on G

Output:

  True if log_G(Y) == log_G(Z), False otherwise

Steps:

1. Signer samples a random element k from Z_q and computes A = kD and B = kE.
2. Signer constructs the challenge c = H_3(D,E,M,Z,A,B).
3. Signer computes s = (k - cx) (mod q).
4. Signer sends (c, s) to the Requestor.
5. Requestor computes A' = (sD + cY) and B' = (sE + cZ).
6. Requestor computes c' = H_3(D,E,M,Z,A',B')
7. Output c == c'.

((TODO: insert explanatory text))

# Tamarin Model

~~~
theory VOPRFs
begin

/*
Protocol:
 Requestor               Signer
     r <-$ G
     M = rH_1(x) 
                   M
                ------->    
                           Z = kM
                   Z
                <-------
    N = Zr^(-1)    
*/

builtins: hashing, asymmetric-encryption, diffie-hellman

/* Generate ephemeral keypair */
rule generate_ephemeral:
   [ Fr(~secretk)] 
   -->
   [ Keys(~secretk, 'g'^~secretk) ]

/* Key Reveals */
rule key_reveal:
   [ Keys(~secretk, 'g'^~secretk) ]--[ Key_reveal(~secretk) ]-> [ Out(~secretk) ]

/* Signed message reveal */
rule Message_reveal:
  let token = ('g'^~secretk)^~x
  in
  [ Finalize(token) ] --[ Reveal_message(token)]-> [ Out(token)]

/* Protocol rules */
rule SendRandomBlindedValue:
  let blinded_input = ('g'^~x)^~r
  in
    [ 
      Fr(~r), // choose random r
      Fr(~x)  // choose random x -- this is supposed to be input to the protocol, though that is not necessary.
    ]
    -->
    [ 
      Out(blinded_input), // output g^{xr}
      StoreBlindedValue(~r, ~x)
    ]

rule SendRandomFixedValue:
  let x = h('voprf')
      blinded_input = ('g'^x)^~r
  in
    [ 
      Fr(~r) // choose random r
    ]
    -->
    [ 
      Out(blinded_input), // output g^{xm}
      StoreBlindedValue(~r, x)
    ]

rule SignAndSendBlindedValue:
    let blinded_value = ('g'^~x)^~r
    in
    [
      Keys(~secretk, 'g'^~secretk),
      In(blinded_value)
    ]
    -->
    [
      Out(blinded_value^~secretk)
    ]

rule UnblindAndFinalizeSignedValue:
  let signed_value = (('g'^~x)^~r)^~secretk
  in
    [ 
      StoreBlindedValue(~r, ~x),
      In(signed_value)
    ]
    --[ Finalize(signed_value^inv(~r)) ]->
    [
      // pass
    ]

// Message secrecy lemma
lemma message_secrecy:
  "(All #i1 t .
    (
      Finalize(t) @ i1 
      &
      not ( (Ex A #ia . Key_reveal( A ) @ ia ) 
          | (Ex B #ib . Reveal_message( B ) @ ib )
          )
    )
    ==> not (Ex #i2. K( t ) @ i2 )
  )"

end
~~~