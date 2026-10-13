---
title: Proving systems
parent: "ADR-0006: Three proving systems"
grand_parent: Rozhodnutia (ADR)
nav_order: 1
---

# Proving systems: analysis of the choice

This page is the background for the proposed
[ADR-0006](../0006-tri-dokazovacie-systemy.md): Groth16, PLONK over
KZG and Marlin. It is also written for readers who do not work with proofs
(indexer, dashboard, deployment). Every term is explained at its first use.

## What a proving system does in our network

A **circuit** is the client's computation written as a system of equations
called **constraints**. The variables are numbers modulo a large prime (such a
set of numbers is called a **field**). The computation is correct exactly when
the values of the variables satisfy all the constraints. The client writes the
circuit, for example in the language Circom, and registers it in the circuit
registry.

The **witness** is the concrete values of all variables of the circuit for one
computation. The miner produces it by running the computation.

A **proof** is a short string of bytes that convinces a verifier that the miner
knows a witness satisfying all the constraints, without sending the witness.
The program that produces the proof is called the **prover** and the program
that checks it the **verifier**.

A **setup** is a step before the first proving. It produces a **proving key**
(needed by the prover) and a **verification key** (needed by the verifier). In
most systems on this page the setup uses secret random numbers that must be
destroyed. Whoever knows them can produce a fake proof. The setup is therefore
done as a **ceremony**: several participants contribute their own randomness,
and the result is secure if at least one of them destroyed their part.

In our network it goes like this:

1. A client sends a task for a registered circuit and pays a fee.
2. The miner computes the witness and the proof. The proof also takes as public
   inputs an integrity value (from the block header) and a mask (from the
   miner's address and the number of uses of the circuit). The integrity value
   prevents moving a proof to another block, and the mask makes the inputs of
   every task unique.
3. The miner puts the proof into a block. Every node verifies every proof in
   every block.

This gives two different costs:

- **Proving time** is paid only by the miner. It is the useful work for which
  the miner receives the reward and lottery tickets. The SNARKChain paper
  (Sec. VI-2) requires it to be predictable from the number of constraints,
  because the fee and the lottery weight are computed from the circuit before
  proving.
- **Proof size and verification time** are paid by the whole network. Every
  node downloads and verifies all proofs of a block. If they grew with the
  circuit size, the capacity of a block would depend on which tasks it
  contains. This is why the paper requires a constant proof size.

Quote from the paper (VI-2): "Alternative proving schemes could be substituted,
provided they maintain constant proof size and their proving time is
predictable from the circuit's constraint count (not necessarily linear)."

## Two axes in which the systems differ

### Axis 1: setup model

- **Setup for each circuit** (Groth16). Every new circuit needs its own
  ceremony. In the paper it is run by the registry nodes together with the
  client (MPC, that is, multi-party computation).
- **Universal SRS** (PLONK, Marlin). The SRS (structured reference string) is a
  large list of elliptic curve points from one ceremony for the whole network.
  The keys for each circuit are then computed from it deterministically
  ("compile" in dusk-plonk, "index" in Marlin). This computation needs no new
  trust, and anyone can repeat it and check it. The Marlin SRS can also be
  updated later by another participant.

The setup axis changes who runs the ceremony and how often, what the registry
stores, and whether **amortization** is possible: proving several tasks
together more cheaply than each separately. Amortization would break the
fairness of the lottery, because a miner would get more tickets for the same
work. The paper (VI-4) says that universal schemes such as PLONK "may support
cross-circuit amortization", while Groth16 keys are bound to a single circuit.
This is exactly what we want to measure.

### Axis 2: arithmetization

**Arithmetization** is the way a computation is written as equations.

- **R1CS** (rank-1 constraint system, Groth16 and Marlin). Each constraint has
  the form (linear combination A) times (linear combination B) equals (linear
  combination C). Circom compiles exactly to R1CS.
- **PLONKish gates** (PLONK). The computation is a table, and each row is one
  gate with selectors that decide whether it adds or multiplies. Further
  constraints (the permutation) say which cells of the table must be equal.
  Custom gates can be added.

The arithmetization axis changes the **complexity measure** from which the fee
is computed. For R1CS it is the number of rows and non-zero entries of the
matrices, for PLONK the number of gates. The same program has a different count
in every arithmetization, so each system needs its own scaling factor.

### Why these three

- All three keep the proof size and the verification time constant (Sec. VI-2,
  Asymmetry in Sec. VI-3).
- Together they cover both setup models the registry has to support (per
  circuit and universal) and both ways of writing circuits (R1CS and PLONKish
  gates), so the prover interface of the node is built for both from the start.
- Marlin takes the same R1CS circuits as Groth16 (from Circom), without a
  ceremony for each circuit.
- All three provers spend most of their time in MSM and FFT over a field of
  about 256 bits, so they run well on the same hardware.

**KZG** is a polynomial commitment: a short fingerprint of a polynomial that
can later be opened at any point with a short proof. It uses the SRS and a
**pairing**, which is an operation on points of two elliptic curves that the
verifier performs a few times. What all three systems have in common is that a
proof has at most about 1 KB and the verifier does a fixed number of pairings
regardless of the circuit size.

## How they work

This part describes what the prover and the verifier of each of the three
systems compute. Detailed derivations are in the sources listed with each
system.

### Groth16

**R1CS by example.** The miner wants to prove that it knows `x` for which
`x^3 + x + 5 = 35`. The computation is split into steps with at most one
multiplication:

```text
s1  = x * x
y   = s1 * x
out = y + x + 5
```

The vector of variables is `a = (1, out, x, s1, y)`. The first component is the
constant 1, `out` is a public input, and the others are known only to the
prover. Each step is one constraint of the form (linear combination) times
(linear combination) equals (linear combination):

| constraint | left factor | right factor | result |
| --- | --- | --- | --- |
| 1 | `x` | `x` | `s1` |
| 2 | `s1` | `x` | `y` |
| 3 | `y + x + 5` | `1` | `out` |

For `x = 3` we have `a = (1, 35, 3, 9, 27)` and `3 * 3 = 9`, `9 * 3 = 27` and
`(27 + 3 + 5) * 1 = 35` hold. The coefficients of the three columns form the
matrices A, B and C with one row per constraint and one column per variable.
Addition costs no constraint, because it hides inside the linear combination.
The number of constraints is the complexity measure from which the paper
computes the fee.

**QAP** (quadratic arithmetic program) writes all constraints as a single
equation of polynomials. Each column of the matrices A, B and C is interpolated
by a polynomial through the points 1 to n, where n is the number of
constraints. The vector `a` thus yields the polynomials `U(X)`, `V(X)` and
`W(X)`. The vector satisfies all constraints exactly when `U(X) * V(X) - W(X)`
is divisible by the polynomial `t(X) = (X - 1)(X - 2)...(X - n)`, that is, when
a quotient `h(X)` exists with `U * V = W + h * t`. Two different polynomials of
low degree agree only in a small number of points, so it is enough to check
this equality at one secret random point `tau`.

**Setup for each circuit.** The setup picks random numbers `tau`, `alpha`,
`beta`, `gamma` and `delta` and computes elliptic curve points from them, for
example the value of each polynomial of the circuit at the point `tau`,
multiplied by a fixed curve point. The number cannot be recovered from such a
point. The points form the proving key (38.5 MiB at 200k constraints) and the
verification key (a few points and one point per public input). The five random
numbers are the **toxic waste**: whoever knows them can produce a valid proof
without a witness. The keys contain the polynomials of a specific circuit, so a
new circuit needs a new setup. The ceremony has two phases. The first (powers
of tau, the powers of `tau`) does not depend on the circuit and can be reused,
and the second is run separately for each circuit.

**Proving.** The prover runs the computation and obtains the witness `a`. It
computes the quotient `h(X)` with six **FFTs** of size n. An FFT (fast Fourier
transform) converts a polynomial between its coefficients and its values at n
points in time proportional to `n log n`. Then it computes three points `A`,
`B` and `C`. Each is an **MSM** (multi-scalar multiplication): a sum
`a_1 * P_1 + ... + a_m * P_m`, where `P_j` are points from the proving key and
`a_j` are witness values or coefficients of `h`. There are four MSMs of about
the size of the number of variables and one of size n, and they make up most of
the prover's time. Two fresh random numbers `r` and `s` blind the proof so that
it does not reveal the witness.

The **proof** is three points: `A` and `C` in the group G1, `B` in the group
G2. G1 and G2 are two groups of points between which the pairing is defined. On
the curve BN254 a compressed G1 point has 32 B and a G2 point 64 B, so the proof
has 128 B (192 B on BLS12-381).

**Verification.** The verifier combines the public inputs into one point
`X = IC_0 + a_1 * IC_1 + ... + a_l * IC_l`, where `IC_j` are points of the
verification key and l is the number of public inputs. This is an MSM of size
l. Then it checks one equation
`e(A, B) = e(alpha, beta) * e(X, gamma) * e(C, delta)`, where `e` is the
pairing and `e(alpha, beta)` is precomputed. That makes 3 pairings. The
verification time does not depend on the circuit size, only linearly on the
number of public inputs (in our case the integrity value, the mask and the
outputs).

Pros:

- the smallest proof and the fastest verification of the known systems,
- mature tooling (Circom, snarkjs, arkworks, gnark) and Circom circuits without
  translation,
- after a change of the public input (the integrity value or the mask) the
  proof must be produced again, and the paper (Sec. IV-C) builds its resistance
  to amortization on this.

Cons:

- a new circuit needs a new ceremony (the second phase),
- the proving key grows with the circuit size,
- the prover is slower than in newer systems,
- the proof is **malleable**: from a valid proof `(A, B, C)` anyone can produce,
  without the witness, another valid proof `(-A, -B, C)` for the same inputs. A
  block must therefore not rely on the proof bytes as an identifier.

Sources: [RareSkills, Groth16 Explained](https://rareskills.io/post/groth16),
[A. Tomescu, Groth16](https://alinush.github.io/groth16),
[J. Groth 2016](https://eprint.iacr.org/2016/260), the SNARKChain paper.

### PLONK: what changes compared with Groth16

- **Arithmetization.** Instead of R1CS the circuit is a table of gates. Every
  row satisfies `qL * a + qR * b + qO * c + qM * a * b + qC = 0`, where `a`, `b`
  and `c` are the values in the row and the `q` are **selectors**, constants of
  the circuit that decide whether the gate adds or multiplies. The example
  above needs four gates (two multiplications, `y + x` and `+ 5`), because here
  an addition costs a gate. Which cells must be equal (the output of the first
  gate is the input of the second) is determined by the **copy constraints**.
  The prover proves them with a permutation argument.
- **Setup.** One universal SRS (powers of a secret number in G1 and one point in
  G2) for all circuits up to a chosen size. The verification key of a circuit
  consists of commitments to the selectors and the permutation (8 G1 points and
  1 G2 point), computed from the SRS without new secret numbers. The SRS is
  secure if at least one participant of the ceremony was honest.
- **Proving.** The prover interpolates the columns of the table into
  polynomials, commits to them via KZG, builds the permutation polynomial and
  the quotient by the domain polynomial (split into three parts) and opens them
  at two points derived from a random challenge. Here too MSM and FFT dominate.
  The PLONK paper estimates 1.1 to 3 times more group operations than Groth16,
  depending on the number of additions. We measured 6.7 times (dusk-plonk on
  BLS12-381 against ark-groth16 on BN254).
- **Proof.** 9 G1 points and 6 field elements in the variant with the faster
  prover (480 B on BN254), 7 points and 6 elements in the smaller variant
  (416 B). dusk-plonk sends more values, and we measured 1008 B on BLS12-381.
- **Verification.** 2 pairings and 16 to 18 point multiplications. The public
  inputs enter through a polynomial that the verifier evaluates itself. This
  work grows with the number of public inputs, but it is only computation in
  the field.

Sources: [PLONK, ePrint 2019/953](https://eprint.iacr.org/2019/953) (Table 1,
Sec. 8.3), [V. Buterin, Understanding PLONK](https://vitalik.eth.limo/general/2019/09/22/plonk.html).

### Marlin: what changes compared with Groth16

- **Arithmetization** stays R1CS, so Circom circuits have the same number of
  constraints as in Groth16.
- **Setup** is a universal SRS as in PLONK and can be updated later by another
  participant. For a circuit the **index** is computed deterministically: the
  non-zero entries of the matrices A, B and C are encoded into polynomials over
  a domain whose size is the number of non-zero entries. The size of the SRS
  therefore determines the number of non-zero entries. In our circuit there are
  about 3 per constraint, so at 200k constraints the domain is 2^20.
- **Proving.** Marlin is an **AHP** (algebraic holographic proof): the verifier
  does not read the matrices of the circuit, only the commitments to the
  polynomials of the index. The prover shows that the matrices multiplied by the
  witness give the correct values, using the **sumcheck** protocol (a proof of
  the sum of the values of a polynomial over the whole domain). Here too MSM
  dominates, making up about 91 % of the prover's time in our port.
- **Proof and verification.** The proof has 855 B (our port, BLS12-381), and the
  verification performs a fixed number of pairings regardless of the circuit
  size (8.67 ms at 200k).

Sources: [Marlin, ePrint 2019/1047](https://eprint.iacr.org/2019/1047),
[README of ark-marlin](https://github.com/arkworks-rs/marlin).

## Measured values at 200,000 constraints

The circuit is a synthetic chain of multiplications (without hashing). One run,
an Intel i7-1260P limited to 4 threads. Groth16 runs on the curve BN254,
dusk-plonk and Marlin on BLS12-381, so the ratios are only indicative.

| system | proving | verification | proof | setup per circuit | proving key |
| --- | --- | --- | --- | --- | --- |
| Groth16 (ark-groth16 0.5) | 8.96 s | 2.37 ms | 128 B | 9.90 s | 38.5 MiB |
| PLONK (dusk-plonk 0.24) | 60.23 s | 5.88 ms | 1008 B | 61.52 s compile | 1200 MiB |
| Marlin (port to arkworks 0.5) | 99.5 s | 8.67 ms | 855 B | 63.99 s index | 515 MiB |
| Spartan 0.9 (for comparison) | 21.87 s | 142.57 ms | 155,632 B | 8.59 s, no ceremony | 584 MiB (decommitment) |

A common SRS for the whole network (generated locally and insecurely, only for
measurement): dusk-plonk 140.13 s and 12 MiB for the domain 2^18, Marlin 23.25 s
for 2^18 constraints and 2^20 non-zero entries.

Notes on the numbers:

- Verification and proof size are constant for all three from 10,000 to 200,000
  constraints. For Spartan both grow (proof 73 to 156 KB, verification 64 to 143
  ms). Growing verification time breaks the paper's argument for Asymmetry,
  which rests on verify time being constant in circuit complexity (SNARKChain,
  arXiv 2510.09729, Sec. III and VI-3). A growing proof size breaks the
  constant-proof-size condition for substitute systems (Sec. VI-2) and raises
  the network overhead that the Efficiency condition (Sec. III) asks to keep
  low.
- Groth16 on 16 threads proves 200k in 2.31 s. The paper reports about 2.6 s
  (Circom and snarkjs, Ryzen 5 7600X), so our measurement is in the same order
  of magnitude.
- Marlin reaches 99.5 s only with our fix of the division in ark-poly (without
  it 140.1 s, because the division in the library is quadratic). With assembly
  in ark-ff (x86_64 only) it is 75.5 s, with arkworks 0.6 and assembly 67.5 s.
  Assembly would speed up the other two systems as well.
- Loading the dusk-plonk proving key (1.2 GB) takes 47.7 s, and loading the SRS
  with point checks 45 s.

What this means for a block (computed from the table, 100 proofs in a block):

| system | verification per node | proofs in a block |
| --- | --- | --- |
| Groth16 | 0.24 s | 12.8 KB |
| PLONK | 0.59 s | 100.8 KB |
| Marlin | 0.87 s | 85.5 KB |
| Spartan | 14.3 s | 15.6 MB |
| STARK (literature) | grows with size (breaks Asymmetry, Sec. VI-3) | 5.1 to 25.2 MB (breaks Sec. VI-2) |

## Who runs the setup

**Groth16 according to the paper.** The ceremony for a new circuit is run at
registration by the nodes of the circuit registry subnet as an MPC. Sec. IV-D:
"SMPC enables nodes to generate the circuit key pair such that security is
preserved as long as at least one node is honest and deletes its randomness
after the ceremony". According to Appendix E the client also contributes input
randomness. The paper does not distinguish the phases of the ceremony.
Normally the first phase is taken over from a public ceremony and only the
second is run for the circuit
([README of snarkjs](https://github.com/iden3/snarkjs): "Any zk-snark project
can pick a round from the common phase 1 to start their circuit-specific phase
2 ceremony").

**PLONK and Marlin.** The paper does not mention them. The options are one MPC
ceremony at the start of the network or an SRS from a public powers of tau
ceremony on the chosen curve. At 200k constraints dusk-plonk needs 2^18 + 10
powers in G1 (the domain 2^18 and 9 extra for blinding), and our port of Marlin
needs 2^20 powers (following the 2^20 non-zero entries). Public ceremonies
(checked 2026-10-13):

| ceremony | curve | largest size | is the size enough at 200k for |
| --- | --- | --- | --- |
| [Perpetual Powers of Tau](https://github.com/privacy-ethereum/perpetualpowersoftau) | BN254 | circuits up to 2^28 constraints | all three |
| [Aztec Ignition](https://github.com/AztecProtocol/ignition-verification) (176 participants, 2019) | BN254 | 100.8 million points | all three |
| [Zcash Powers of Tau](https://github.com/ebfull/powersoftau) | BLS12-381 | 2^22 - 1 powers in G1 | all three |
| [Filecoin](https://github.com/filecoin-project/phase2-attestations) | BLS12-381 | 2^27 (according to the phase 1.5 files) | all three |
| [Ethereum KZG](https://github.com/ethereum/kzg-ceremony-specs) (EIP-4844) | BLS12-381 | 4096 to 32,768 powers in G1 | none |

The SRS in ark-poly-commit (Marlin) has public fields, so it can be assembled
from the points of a ceremony. dusk-plonk loads the SRS only in its own byte
format and would need a converter. We have tried neither yet.

**Cost.** We measured only generation by a single party (4 threads): the
Groth16 keys for a 200k circuit 9.90 s, the dusk-plonk SRS for the domain 2^18
140.13 s, the Marlin SRS for 2^20 non-zero entries 23.25 s. The dusk-plonk
generator computes every power with a separate point multiplication, so with 4
times fewer powers it takes 6 times longer than the Marlin generator. An MPC
ceremony is a sequence of contributions, and each costs roughly one generation
plus checking the previous contribution. An estimate, not a measurement: with
10 participants the second phase of Groth16 for a 200k circuit takes several
minutes at every registration, and the one-time SRS ceremony takes from several
minutes (Marlin) to tens of minutes (dusk-plonk).

**Curve.** We measured Groth16 on BN254, and dusk-plonk and Marlin on
BLS12-381. A common ceremony or a common public SRS requires one curve for the
whole network. The choice of the curve is a separate decision for a further
ADR.

## Why not AIR, STARK and zkVM

**AIR** (algebraic intermediate representation) writes a computation as a table
with a fixed number of columns (the trace), where the same transition rule holds
between every two adjacent rows. **STARK** is a proving system over AIR that
uses only hash functions and needs no setup. A **zkVM** is a virtual processor
(for example RISC-V): the program is compiled for this processor and its
execution is proved. Examples are winterfell, Plonky3, stwo, SP1, RISC Zero and
Miden.

1. **Price.** In R1CS and PLONK the complexity measure is a fixed count in the
   circuit file. In AIR it is the length times the width of the trace, in a zkVM
   the number of cycles. The number of cycles is a property of one run, not of
   the program: it is known only after execution, changes with the input and
   depends on the compiler. A fee determined before the run is an estimate, and
   a fee determined after the run is set by the miner. A cycle limit is also
   needed, which is another pricing decision. The conversion from cycles to fee
   has no source.
2. **Hardware imbalance.** Pairing-based provers mostly compute MSM and FFT over
   a 256-bit field. STARK provers mostly compute hashes over small fields and
   are accelerated by different hardware. If miners choose their tasks, every
   error in the exchange rate between systems sends specialized hardware to the
   better paid system, and the tasks of the other system stay unproved. If the
   protocol assigns the tasks, the imbalance becomes variance: a miner with
   unsuitable hardware proves slowly and loses the lottery through no fault of
   its own. Assignment also needs verifiable randomness and a stake, and does not
   solve the exchange rate between systems.
3. **Integration.** AIR has no path from R1CS, so Circom circuits would have to
   be rewritten. A second circuit language would appear, a second witness model
   (a trace instead of an assignment of variables) and a second field
   (Goldilocks, BabyBear, Mersenne-31), so the integrity value and the mask
   would need a second encoding. A zkVM proves a long program in parts and joins
   the parts, which conflicts with the rule of one proof per transaction.
4. **Block cost.** Proofs of 51 to 252 KB (winterfell, literature, Miden 154 to
   192 KiB) instead of 128 B to 1 KB, and a verification that grows with the
   size of the computation (breaks Asymmetry, Sec. VI-3, and the proof size
   breaks Sec. VI-2). Every node pays this, and it does not satisfy VI-2.
5. **Maturity.** The libraries change quickly: Plonky3 has a stable version
   0.8.0 and already 0.9.0-rc.1, Miden VM is in the alpha state according to its
   README, and SP1 and RISC Zero change the proof format between major versions.

We acknowledge the strengths of STARKs: no setup, resistance to quantum
computers and the fastest prover. If the team wants a system without a setup, it
can be added as a fourth one once the node architecture is stable. It is not
meant to replace any of the three.

Other rejected options:

- **Spartan** and other transparent systems over R1CS (Aurora, Fractal, Ligero,
  Brakedown): the proof grows with the circuit size (breaks Sec. VI-2).
  Spartan is the fallback if the team drops the constant-size condition.
- **PLONK over IPA** (halo2 from Zcash): no SRS, but the proof grows as a
  logarithm and the verification linearly (breaks Sec. VI-2 and Asymmetry, Sec.
  VI-3).
- **Plonky2** (PLONK over FRI): proofs of the size of a STARK.
- **UltraHonk, HyperPlonk, Gemini**: proofs grow as a logarithm, at the edge of
  the condition (strictly speaking breaks the constant-size condition, Sec. VI-2). UltraHonk exists only in
  Barretenberg behind the language Noir.
- **Folding** (Nova, HyperNova, ProtoStar): meant for incremental computations,
  and for a standalone proof another SNARK is needed at the end anyway.
- **Bulletproofs**: linear verification (breaks Asymmetry, Sec. VI-3).

## Weaknesses of the choice

- **Narrowness.** All three are pairing-based SNARKs with an SRS on BN254 or
  BLS12-381. A reader of the assignment may expect a transparent system. The
  list in the assignment is an example, the rule of the paper VI-2 excludes
  proofs of the size of a STARK, and both axes on which the node design depends
  are covered. The rejection is recorded on this page.
- **Cost of the Marlin prover.** About 11 times Groth16 and 1.7 times
  dusk-plonk at 200k. After the fix of the division, MSM makes up about 91 % of
  the time. Tasks for Marlin will be expensive for miners, and the scaling factor
  of the lottery weight must take that into account.
- **Our own fork of Marlin.** Upstream ark-marlin targets arkworks 0.3 and its
  last commit is from 2022-08-29. The release 0.3.0 does not build on the
  current nightly Rust compiler. Our fork has two commits: a port to arkworks 0.5
  (10 files, 215 added and 196 removed lines) and a linear division in the
  prover (53 added lines). Together it adds 249 and removes 179 lines in a
  source tree of 2,971 lines, and it passes the tests of the original library.
  Maintenance means carrying these changes over to each new arkworks release.
  The move to arkworks 0.6 (branch `speedup-0.6`) took 59 added and 45 removed
  lines. The Fiat-Shamir transcript (a hash from which the prover derives the
  verifier's challenges, so that the proof needs no interaction) is used in the
  port also for the challenges when opening commitments. Nobody has reviewed
  this change for security yet. The connection to ark-circom is not verified
  yet.
- **SRS and keys.** Both universal systems need one SRS ceremony for the
  network, with a domain at least as large as the largest circuit. The proving
  keys at 200k are 1.2 GB (dusk-plonk) and 515 MiB (Marlin) against 38.5 MiB for
  Groth16. The registry either stores the proving keys, or only the verification
  keys and each miner recomputes the proving key (61.5 s or 64.0 s per circuit).

## Fallback options

- **Marlin does not accept the Circom output.** A fallback with the same meaning
  (universal setup over R1CS): Varuna in snarkVM, derived from Marlin, licensed
  Apache-2.0, actively developed. We have not yet verified its prover entry
  point. A fallback with a different meaning: UltraHonk via Barretenberg, which
  changes the circuit language to Noir and the proof size to logarithmic.
- **The team rejects a third pairing-based system.** Fallback: Spartan, already
  built and measured, transparent, over R1CS. It adds to the setup axis an
  option without a setup, at the cost of the constant proof size.

## Open checks

- ark-marlin with input from ark-circom: does an adapter exist, or can one be
  built?
- The prover entry point of Varuna and its input format.
- Marlin and dusk-plonk on the same curve as Groth16 (or Groth16 on BLS12-381),
  so that the ratios are comparable.
- Repeated runs of all measurements, which are one-off now.
- Loading the SRS from a public ceremony into ark-poly-commit and dusk-plonk
  (see the section Who runs the setup).

## State of the libraries

The comparison of libraries (supported systems, maturity, documentation,
license, circuit notation) is on the page
[Proof libraries](kniznice-pre-dokazy.md). For this choice it says:

- **Groth16**: ark-groth16 0.6 with ark-circom loads Circom circuits (`.r1cs`
  and `.wasm`) directly. We measured version 0.5.
- **PLONK over KZG**: dusk-plonk is actively developed and audited, but
  according to its README "currently unstable", and licensed MPL-2.0. It does
  not read Circom circuits. halo2 from PSE is archived, and its continuation
  halo2-axiom has a different API.
- **Marlin**: upstream ark-marlin has its last commit from 2022-08-29, so we use
  our fork (see Weaknesses of the choice). Varuna in snarkVM is an actively
  developed fallback, and we have not verified its input for a custom R1CS.
- **Barretenberg** in the current `bb` CLI offers only UltraHonk for Noir, so it
  is not a fallback for PLONK over KZG with a constant proof.
