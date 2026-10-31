---
title: Rozhodnutia (ADR)
nav_order: 6
has_children: true
---

# Architektonické rozhodnutia

Každé dôležité technické rozhodnutie zapíšeme ako ADR, aby sme sa k nemu nevracali.

1. Skopíruj [šablónu](TEMPLATE.md) do súboru `NNNN-kratky-nazov.md` (napríklad `0001-kniznice-pre-dokazy.md`).
2. Vyplň ho a otvor pull request.
3. Po schválení tímom zmeň stav na **prijaté**.

## Navrhnuté rozhodnutia

| ADR | Rozhodnutie |
| --- | --- |
| [ADR-0006](0006-tri-dokazovacie-systemy.md) | Groth16, PLONK over KZG and Marlin as the three proving systems |
| [ADR-0007](0007-ulozene-a-odvodene-polia.md) | stored and derived fields of Block and ProofTx |
| [ADR-0008](0008-indexer-a-explorer.md) | indexer and explorer, reuse or build (open for discussion) |

## Prijaté rozhodnutia

| ADR | Rozhodnutie |
| --- | --- |
| [ADR-0001](0001-jenkins-pre-testy-celej-siete.md) | Jenkins pre testy celej siete vedľa GitHub Actions |
| [ADR-0002](0002-izolovany-docker-pre-buildy.md) | buildy Jenkinsu v izolovanom Dockeri |
| [ADR-0003](0003-hesla-v-pass.md) | heslá v `pass`, do kontajnerov ako súbory |
| [ADR-0004](0004-vecerny-deploy-na-staging.md) | večerné nasadenie na staging iba po úspešných testoch |
| [ADR-0005](0005-https-cez-caddy.md) | HTTPS cez Caddy a sslip.io, ostatné porty iba lokálne |
| [ADR-0011](0011-adr-vstupy-v-bloku.md) | witness inputs travel inside the proof request and are stored in the block |
| [ADR-0010](0010-adr-lahky-klient.md) | light client as a Rust crate of pouw-node, used by the workload generator |
