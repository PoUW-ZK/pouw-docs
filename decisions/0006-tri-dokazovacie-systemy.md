---
title: "ADR-0006: Three proving systems"
parent: Rozhodnutia (ADR)
nav_order: 6
has_children: true
---

# ADR-0006: Groth16, PLONK over KZG and Marlin as the three proving systems

**Status:** proposed
**Date:** 2026-10-13
**Author:** @Ddystopia

## Context

The assignment asks for one blockchain that accepts proofs from several proving
systems, and gives "Groth16, PLONK and STARK-type systems" as an example. On
2026-10-12 the team confirmed one chain with three systems. Groth16 is fixed by
the SNARKChain paper, so two more must be chosen.

A proving system is the way a miner produces a short proof that it correctly
ran the client's computation (the circuit), and the way other nodes check that
proof without repeating the computation. In our network every node verifies
every proof in every block. The verification time and the proof size are
therefore paid by the whole network, while the proving time is paid by the
miner.

Three things constrain the choice:

- The SNARKChain paper ([arXiv 2510.09729](https://arxiv.org/abs/2510.09729),
  Sec. VI-2) allows a substitute system if it has a constant proof size and a
  proving time that is predictable from the number of constraints of the
  circuit.
- The fee and the lottery weight are computed from a complexity measure of the
  circuit before proving. The measure must therefore be readable from the
  circuit file.
Terms used below:

- **Setup** is a one-time step that produces the prover key and the verifier
  key from secret random numbers. Whoever knows those numbers can forge proofs.
- **SRS** (structured reference string) is the result of one setup that can be
  used for any circuit up to a certain size (a universal setup).
- **Arithmetization** is the way a computation is written as equations.
  **R1CS** writes each constraint as a product of two linear expressions equal
  to a third one (this is what Circom compiles to). **PLONKish** writes the
  computation as a table of gates.
- **KZG** is a commitment scheme for polynomials that both PLONK and Marlin use
  to get short proofs. **Pairing-based SNARKs** verify a proof with a few
  pairings, which are operations on points of elliptic curves.
- **MSM** and **FFT** are the two most expensive operations of these provers:
  a sum of many multiples of curve points, and a fast transform of
  polynomials.
- **Amortization** means proving several tasks together more cheaply than each
  one separately. A miner would then get more lottery tickets for the same
  work.

The detailed analysis and measurements are on the page
[Proving systems](0006/dokazovacie-systemy.md), the comparison of
libraries is on the page
[Proof libraries](0006/kniznice-pre-dokazy.md).

## Options

All options contain Groth16 and PLONK. They differ in the third system.
Pros and cons are focused on either the third option or how it integrates
with the other two.

1. **Groth16 + PLONK over KZG + Marlin**
   - pros: every proof is at most 1008 B and is verified in 2.4 to 8.7 ms
     regardless of the circuit size. All three run on the same
     hardware, and Groth16 and Marlin both take R1CS, so circuits from Circom
     work.
   - cons: all three are pairing-based SNARKs, so there is no system without a
     setup. The Marlin prover is about 11 times slower than Groth16 at 200k
     constraints. Marlin runs on our small fork of ark-marlin (a port to
     arkworks 0.5 and a fix of the division, about 270 added and 200 removed
     lines).
2. **Groth16 + PLONK + STARK** (the example from the assignment)
   - pros: no setup, resistance to quantum computers, the fastest prover.
   - cons: proofs of 51 to 252 KB and verification that grows with the size of
     the computation (this breaks the paper's argument for Asymmetry, which
     rests on verify time being constant in circuit complexity, Sec. III and
     VI-3. The large proofs break the constant-proof-size condition of Sec.
     VI-2 and raise the network overhead that the Efficiency condition, Sec.
     III, asks to keep low). The complexity is known only after the program
     runs, so the fee cannot be determined in advance. Different hardware,
     a different circuit language and a different field.
3. **Groth16 + PLONK + Spartan**
   - pros: no setup, takes R1CS, already built and measured.
   - cons: the proof is 156 KB at 200k constraints and grows as the square
     root of the number of constraints, the verification of 143 ms grows
     linearly (breaks Asymmetry, Sec. VI-3, and the proof size breaks Sec.
     VI-2).
4. **Groth16 + PLONK + UltraHonk via Barretenberg**
   - pros: actively developed, a fast prover in C++.
   - cons: C++ through FFI and the circuit language Noir. Among the proving
     systems for Noir, the current `bb` CLI offers only UltraHonk, whose
     proofs grow as the logarithm of the circuit size (breaks Sec. VI-2). It
     is again a PLONKish system with a universal SRS, so it would be a second
     variant of PLONK rather than a different kind of system.

## Decision

**Proposed**, the team still has to approve it: Groth16 (crate ark-groth16), PLONK
over KZG (crate dusk-plonk) and Marlin (a port of ark-marlin to arkworks 0.5).

Where this choice wins:

- **Block.** A proof has 128 to 1008 B and verification takes constant time.
  The verification budget of a block is one number per system and a proof
  transaction has a fixed maximum size. With 100 proofs in a block this is at
  most 0.9 s of verification per node, while with STARK proofs the proofs alone
  would take 5 to 25 MB.
- **Price.** The complexity measure is the number of rows, non-zero entries or
  gates, directly in the circuit file. Registration can determine the fee
  before proving.
- **Hardware.** All three provers mostly compute MSM and FFT over a 256-bit
  field. An error in the exchange rate between systems therefore does not move
  the miners' performance to one system.
- **Input.** Circom circuits (the format of the paper) go to Groth16 directly
  and in principle to Marlin as well (not verified yet).

## Consequences

- The node will be generic over three systems. All three verifiers are
  pairing-based, so the verification code and the measurement of block time
  have the same shape.
- The circuit registry needs two setup modes: a ceremony for each circuit
  (Groth16) and one SRS for the whole network (PLONK, Marlin), from our own
  ceremony or from a public one. A public ceremony fixes the curve, and its
  choice will be a separate ADR.
- We need to decide whether the registry stores proving keys (up to 1.2 GB per
  circuit at 200k constraints) or only verification keys, and each miner
  recomputes the proving key (61.5 s for PLONK, 64.0 s for Marlin).
- We take Marlin from our fork of ark-marlin, because the last upstream commit
  is from 2022-08-29. The fork adds 249 and removes 179 lines in a source tree
  of 2,971 lines (a port to arkworks 0.5 and a linear division). Maintenance
  means carrying these changes over to each new arkworks release. The move to
  arkworks 0.6 took 59 added and 45 removed lines. The port uses a Fiat-Shamir
  transcript (a hash from which the prover derives the verifier's random
  challenges) also for the challenges when opening commitments. Nobody has
  reviewed this change for security yet.
- If Marlin does not accept circuits from Circom, the fallback is Varuna from
  snarkVM. If the team rejects a third pairing-based system, the fallback is
  Spartan.

## Illustrative example tasks

The following examples are not the reason for the choice. They show what
useful work fits the properties of each system.

- **Groth16: authenticity of photographs.** A camera signs an image and the
  proof shows that the published picture was derived from it only by allowed
  edits, for example cropping or downscaling. The circuit is fixed and used
  very often, so a ceremony for a single circuit pays off, and the smallest
  proof (128 B) saves space in every block. Similar works:
  [PhotoProof](https://www.cs.tau.ac.il/~tromer/papers/photoproof-oakland16.pdf),
  [ZK-IMG](https://arxiv.org/abs/2211.04775).
- **PLONK over KZG: inference of small ML models (zkML).** Every new or
  retrained model is a new circuit. The universal SRS allows adding a circuit
  without a new ceremony, and PLONKish gates describe the tabular operations
  of models well. Similar works: [EZKL](https://github.com/zkonduit/ezkl)
  (halo2 with KZG), [Kang et al. 2022](https://arxiv.org/abs/2210.08674).
- **Marlin: per-event circuits in Circom.** For example private voting or an
  audit check, where each event has different parameters and therefore a
  different circuit. A ceremony for each circuit would be impractical, and the
  universal SRS over R1CS allows using existing Circom circuits without one.
  A similar project: [MACI](https://github.com/privacy-ethereum/maci)
  (circuits in Circom, the repository is archived).
