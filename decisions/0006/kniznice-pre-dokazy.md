---
title: Proof libraries
parent: "ADR-0006: Three proving systems"
grand_parent: Rozhodnutia (ADR)
nav_order: 2
---

# Proof libraries: overview

This page compares the libraries and circuit languages from which the prover
and the verifier of the node can be built. The choice of the three systems is in
[ADR-0006](../0006-tri-dokazovacie-systemy.md). The explanation of
terms (circuit, R1CS, PLONKish, SRS, setup) and the measurements are on the page
[Proving systems](dokazovacie-systemy.md).

Further terms:

- **AIR** (algebraic intermediate representation) writes a computation as a
  table (the trace) where the same transition rule holds between every two
  adjacent rows. It is used by **STARK** systems, which need no setup.
- **IPA** (inner product argument) is a commitment scheme for polynomials
  without an SRS. The proof grows as the logarithm of the circuit size and the
  verification grows linearly (breaks the constant-proof-size condition of the
  SNARKChain paper, arXiv 2510.09729, Sec. VI-2, and its argument for
  Asymmetry, which rests on constant verify time, Sec. III and VI-3).
- **Varuna** is a system derived from Marlin in the snarkVM library of the Aleo
  project.
- **Noir** is a circuit language that compiles to the ACIR format. The prover
  **Barretenberg** (the command `bb`) produces proofs from it.
- **Maturity** here means the latest release, the date of the last commit on the
  main branch, and what the README says about the state (audit, warning against
  production use).

Versions, dates and licenses were checked on 2026-10-13 via the crates.io API,
the npm registry and the GitHub API. The description of circuit notation comes
from the official READMEs and tutorials, and the examples are below the table.

## Comparison

| library | supported systems | maturity | documentation | license | how a circuit is written |
| --- | --- | --- | --- | --- | --- |
| ark-groth16 (arkworks, Rust) | Groth16 over R1CS | 0.6.0 (2026-04-26), commit 2026-08-02. README: academic prototype, not for production | docs.rs, [r1cs-tutorial](https://github.com/arkworks-rs/r1cs-tutorial) | MIT OR Apache-2.0 | Rust, trait `ConstraintSynthesizer` |
| ark-marlin (arkworks, Rust) | Marlin over R1CS, universal SRS | 0.3.0 (2021-06-07), last commit 2022-08-29. Our fork on arkworks 0.5 without a release | README and docs.rs | MIT OR Apache-2.0 | like ark-groth16 |
| ark-circom + Circom | Groth16 (ark-circom), Circom produces R1CS and a witness generator | ark-circom 0.6.0 (2026-04-26), circom v2.2.3 (2025-10-27), circom commit 2026-03-16 | [docs.circom.io](https://docs.circom.io) | ark-circom MIT OR Apache-2.0, the circom compiler GPL-3.0 | language Circom, ark-circom loads `.r1cs` and `.wasm` at runtime |
| halo2_proofs (Zcash, Rust) | PLONKish over IPA, no setup | 0.4.0 (2026-09-29), commit 2026-10-09 | [halo2 book](https://zcash.github.io/halo2/) | MIT OR Apache-2.0 | Rust, trait `Circuit` with columns, gates and chips |
| halo2 PSE and halo2-axiom (Rust) | PLONKish over KZG | PSE v0.4.0 (2024-09-13), repository archived, maintenance only since 2025-01. halo2-axiom 0.5.3 (2026-08-14), audit only for tagged releases | halo2 book, [docs.axiom.xyz](https://docs.axiom.xyz) | MIT OR Apache-2.0 | like halo2_proofs |
| dusk-plonk (Rust) | PLONK over KZG, BLS12-381 | 0.24.0 (2026-10-08), commit 2026-10-11. README: "currently unstable", audit done | docs.rs, folder `examples` | MPL-2.0 | Rust, trait `Circuit`, gates via `Composer` |
| winterfell (Rust) | STARK over AIR | 0.13.1 (2025-07-19), no commits since. README: research project, no audit | tutorial in the README, docs.rs | MIT | Rust, trait `Air` with transition constraints |
| Plonky3 (Rust) | toolkit for STARK over AIR (FRI and others) | p3-uni-stark 0.8.0 (2026-09-23), 0.9.0-rc.1 (2026-10-11), releases every 2 to 3 weeks. README makes no statement about an audit | README and examples, docs.rs | MIT OR Apache-2.0 | Rust, `BaseAir` and `Air` |
| Spartan (libspartan, Rust) | Spartan over R1CS, no setup | 0.9.0 (2024-12-20), commit 2026-09-14. README: no security review and no audit | README, paper, docs.rs | MIT (crates.io lists its own license file) | matrices A, B, C given as a list of non-zero entries |
| snarkVM, Varuna (Rust) | Varuna over R1CS, universal SRS | 4.11.0 (2026-10-01), commit 2026-10-13. README makes no statement about an audit | [docs.leo-lang.org](https://docs.leo-lang.org), docs.rs | Apache-2.0, the Leo compiler GPL-3.0 | language Leo or Aleo instructions. Input for a custom R1CS not verified |
| gnark (Go) | Groth16 and PLONK, BN254, BLS12-381, BLS12-377 | v0.16.4 (2026-10-07), several audits | [`docs.gnark.consensys.net`](https://docs.gnark.consensys.net) | Apache-2.0 | Go, a struct with a `Define` method |
| bellman (Rust) | Groth16 over R1CS | 0.15.0 (2026-09-26), before that 0.14.0 from 2023. README makes no statement about an audit | example in the crate documentation, docs.rs | MIT OR Apache-2.0 | Rust, trait `Circuit` |
| Barretenberg and Noir (C++, Rust) | UltraHonk (proof grows as a logarithm, breaks Sec. VI-2), the `bb` CLI has the schemes `ultra_honk`, `chonk` and `avm` | aztec-packages v5.2.1 (2026-10-08), Noir v1.0.0-rc.4 (2026-10-06). Noir README: no audit, not for production | [noir-lang.org/docs](https://noir-lang.org/docs/) | Apache-2.0 | language Noir |
| snarkjs and rapidsnark (JavaScript, C++) | snarkjs: Groth16, PLONK, FFLONK (beta). rapidsnark: Groth16 prover | snarkjs 0.7.6 (npm, 2026-01-12), commit 2026-10-01. rapidsnark v0.0.8 (2026-01-07) | tutorial in the snarkjs README | snarkjs GPL-3.0, rapidsnark LGPL-3.0 | Circom circuits (`.r1cs`), command-line tool |

## How a circuit is written

Examples are shortened excerpts from official sources, with a link at each one.

### arkworks and bellman (R1CS in Rust)

A circuit is a Rust type that allocates variables and adds constraints during
synthesis. The same code serves both ark-groth16 and ark-marlin. Source:
[`groth16/tests/mimc.rs`](https://github.com/arkworks-rs/groth16/blob/master/tests/mimc.rs).

```rust
impl<'a, F: PrimeField> ConstraintSynthesizer<F> for MiMCDemo<'a, F> {
    fn generate_constraints(self, cs: ConstraintSystemRef<F>) -> Result<(), SynthesisError> {
        let mut xl = FpVar::new_witness(cs.clone(), || {
            self.xl.ok_or(SynthesisError::AssignmentMissing)
        })?;
        let output = FpVar::new_input(cs.clone(), || {
            self.output.ok_or(SynthesisError::AssignmentMissing)
        })?;
        // ... MiMC rounds over xl and xr
        output.enforce_equal(&xl)?;
        Ok(())
    }
}
```

bellman works on the same principle: a trait `Circuit` with a method
`synthesize` that allocates variables through `ConstraintSystem`.

### Circom, snarkjs and ark-circom

A circuit is written in the language Circom, where the operator `<==` assigns a
value and adds a constraint at the same time. Source:
[Writing circuits](https://docs.circom.io/getting-started/writing-circuits/).

```circom
pragma circom 2.0.0;

template Multiplier2 () {
   signal input a;
   signal input b;
   signal output c;
   c <== a * b;
}
```

The compiler produces `.r1cs` and a witness generator `.wasm`. snarkjs uses
them to run the setup and the proof from the command line
([README of snarkjs](https://github.com/iden3/snarkjs)):

```sh
snarkjs groth16 setup circuit.r1cs pot14_final.ptau circuit_0000.zkey
snarkjs zkey contribute circuit_0000.zkey circuit_0001.zkey --name="1st Contributor Name" -v
snarkjs groth16 prove circuit_final.zkey witness.wtns proof.json public.json
snarkjs groth16 verify verification_key.json public.json proof.json
snarkjs plonk setup circuit.r1cs pot14_final.ptau circuit_final.zkey
```

rapidsnark replaces the step `groth16 prove`. ark-circom loads the same files in
Rust (`CircomConfig::new(wasm, r1cs)`, then `CircomBuilder`) and proves via
ark-groth16 ([README](https://github.com/arkworks-rs/circom-compat)).

### halo2

A circuit declares the columns of the table in `configure` and fills them via a
`Layouter` in `synthesize`. The gates are in a separate chip (`FieldChip`).
Source:
[simple-example.rs](https://github.com/zcash/halo2/blob/main/halo2_proofs/examples/simple-example.rs).

```rust
fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config {
    let advice = [meta.advice_column(), meta.advice_column()];
    let instance = meta.instance_column();
    let constant = meta.fixed_column();
    FieldChip::configure(meta, advice, instance, constant)
}

fn synthesize(&self, config: Self::Config, mut layouter: impl Layouter<F>) -> Result<(), Error> {
    let field_chip = FieldChip::<F>::construct(config);
    let a = field_chip.load_private(layouter.namespace(|| "load a"), self.a)?;
    let b = field_chip.load_private(layouter.namespace(|| "load b"), self.b)?;
    let ab = field_chip.mul(layouter.namespace(|| "a * b"), a, b)?;
    // ... absq = ab * ab, c = constant * absq
    field_chip.expose_public(layouter.namespace(|| "expose c"), c, 0)
}
```

### dusk-plonk

A circuit adds gates via `Composer`. Each gate has selectors (`left`, `right`,
`mult`, `constant`) and wires `a`, `b`, `d`. Source:
[examples/circuit.rs](https://github.com/dusk-network/plonk/blob/master/examples/circuit.rs).

```rust
impl Circuit for TestCircuit {
    fn circuit(&self, composer: &mut Composer) -> Result<(), Error> {
        let a = composer.append_witness(self.a);
        let b = composer.append_witness(self.b);
        // ... more variables and a range check
        let constraint = Constraint::new()
            .left(1).right(1).a(a).b(b).constant(BlsScalar::from(42));
        let result = composer.gate_add(constraint);
        let c = composer.append_public(self.c);
        composer.assert_equal(result, c);
        // ... more gates
        Ok(())
    }
}
```

### winterfell and Plonky3 (AIR)

The computation is a trace and the circuit is a rule that must hold between a
row and the next row, plus conditions for the first and the last row.
winterfell
([README](https://github.com/facebook/winterfell)):

```rust
fn evaluate_transition<E: FieldElement + From<Self::BaseField>>(
    &self, frame: &EvaluationFrame<E>, _periodic_values: &[E], result: &mut [E],
) {
    let current_state = frame.current()[0];
    let next_state = current_state.exp(3u32.into()) + E::from(42u32);
    result[0] = frame.next()[0] - next_state;
}
```

Plonky3 writes the same through `AirBuilder`
([uni-stark/tests/fib_air.rs](https://github.com/Plonky3/Plonky3/blob/main/uni-stark/tests/fib_air.rs)):

```rust
fn eval(&self, builder: &mut AB) {
    let main = builder.main();
    let local: &FibonacciRow<AB::Var> = main.current_slice().borrow();
    let next: &FibonacciRow<AB::Var> = main.next_slice().borrow();
    let mut when_transition = builder.when_transition();
    when_transition.assert_eq(local.right, next.left);
    when_transition.assert_eq(local.left + local.right, next.right);
}
```

### Spartan

libspartan has no circuit language. The matrices A, B, C are given as a list of
non-zero entries (row, column, value). Source:
[examples/cubic.rs](https://github.com/microsoft/Spartan/blob/master/examples/cubic.rs).

```rust
// constraint 0 is Z0 * Z0 - Z1 = 0.
A.push((0, 0, one));
B.push((0, 0, one));
C.push((0, 1, one));
// constraint 1 is Z1 * Z0 - Z2 = 0.
A.push((1, 1, one));
B.push((1, 0, one));
C.push((1, 2, one));
let inst = Instance::new(num_cons, num_vars, num_inputs, &A, &B, &C).unwrap();
```

### snarkVM (Leo)

A program in the language Leo is translated to Aleo instructions and snarkVM
proves it via Varuna. Source:
[leo-examples/helloworld](https://github.com/ProvableHQ/leo-examples/blob/main/helloworld/src/main.leo).

```leo
program helloworld.aleo {
    fn main(public a: u32, b: u32) -> u32 {
        return a + b;
    }
}
```

### gnark

A circuit is a struct in Go, and public variables carry the tag
`gnark:",public"`. The example from the README is the same computation as on the
page [Proving systems](dokazovacie-systemy.md). Source:
[README of gnark](https://github.com/Consensys/gnark).

```go
type CubicCircuit struct {
    X frontend.Variable `gnark:"x"`
    Y frontend.Variable `gnark:",public"`
}

func (circuit *CubicCircuit) Define(api frontend.API) error {
    x3 := api.Mul(circuit.X, circuit.X, circuit.X)
    api.AssertIsEqual(circuit.Y, api.Add(x3, circuit.X, 5))
    return nil
}
```

### Noir a Barretenberg

A program in the language Noir, where `pub` marks public inputs. `nargo`
computes the witness and `bb` produces and verifies the proof. Source:
[Noir docs](https://noir-lang.org/docs/getting_started/quick_start/).

```sh
# src/main.nr: fn main(x: u32, y: pub u32) -> pub u32 { x + y }
nargo execute
bb prove -b ./target/hello_world.json -w ./target/hello_world.gz --write_vk -o target
bb verify -p ./target/proof -k ./target/vk
```

## What remains unverified

- External converters from Circom for bellman and gnark.
- Audits of the libraries bellman, snarkVM, Plonky3 and Barretenberg (their
  READMEs do not mention them).
- The prover entry point of Varuna for a custom R1CS, and an example in Aleo
  instructions.
- Whether `r1cs-tutorial` works with arkworks 0.6.
