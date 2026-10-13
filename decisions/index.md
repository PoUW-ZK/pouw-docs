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

## Prijaté rozhodnutia

| ADR | Rozhodnutie |
| --- | --- |
| [ADR-0001](0001-jenkins-pre-testy-celej-siete.md) | Jenkins pre testy celej siete vedľa GitHub Actions |
| [ADR-0002](0002-izolovany-docker-pre-buildy.md) | buildy Jenkinsu v izolovanom Dockeri |
| [ADR-0003](0003-hesla-v-pass.md) | heslá v `pass`, do kontajnerov ako súbory |
| [ADR-0004](0004-vecerny-deploy-na-staging.md) | večerné nasadenie na staging iba po úspešných testoch |
| [ADR-0005](0005-https-cez-caddy.md) | HTTPS cez Caddy a sslip.io, ostatné porty iba lokálne |
