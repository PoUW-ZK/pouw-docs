---
title: "ADR-0001: Jenkins pre testy celej siete"
parent: Rozhodnutia (ADR)
nav_order: 1
---

# ADR-0001: Jenkins pre testy celej siete vedľa GitHub Actions

**Stav:** prijaté
**Dátum:** 2026-10-12
**Autor:** @Faustynn

## Kontext

Každá služba (uzol, indexer, generátor záťaže, dashboard) má vlastný
repozitár a vlastné CI v GitHub Actions (`ci-ok`): lint, unit testy, audit
závislostí. To overí službu samu osebe, ale nie to, či služby fungujú spolu.

Potrebujeme:

- postaviť všetky služby z `main` a spustiť ich spolu ako jednu sieť;
- pustiť na nej testy celej siete a vidieť, ktoré prešli, ktoré nie a odkedy;
- robiť to každý večer od nuly, aby sme zachytili aj zmeny v závislostiach;
- po úspešných testoch nasadiť presne tie verzie na staging.

## Možnosti

1. **Iba GitHub Actions** (plánovaný workflow, sieť v Docker Compose na runneri)
   - výhody: nič ďalšie na prevádzku, všetko na jednom mieste;
   - nevýhody: celá sieť beží na cudzom runneri s obmedzenou pamäťou a časom;
     história testov je iba v logoch a artefaktoch, bez prehľadu „odkedy test
     padá“; nasadenie na náš server z cudzieho stroja.
2. **Jenkins na vlastnom serveri**
   - výhody: vlastný hardvér vedľa stagingu; záložka Tests s históriou a
     trendom; multibranch joby pre vetvy a PR; výsledok ide ako status do PR;
   - nevýhody: ďalšia služba na údržbu a zabezpečenie, Jenkins musí byť
     dostupný zvonka kvôli webhookom.
3. **Iný CI server** (GitLab CI, Drone, Woodpecker)
   - výhody: modernejšia konfigurácia;
   - nevýhody: kód je na GitHube, integrácia by bola horšia a nikto v tíme ich
     nepoužíva.

## Rozhodnutie

Rozdelenie podľa účelu:

- **GitHub Actions** ostáva pre rýchle kontroly jednotlivých repozitárov
  (`ci-ok`, povinné pre merge) a pre zostavenie Docker obrazov.
- **Jenkins** na staging serveri robí test celej siete, večerný build od nuly
  a nasadenie na staging. Pre `pouw-deploy` je jeho výsledok tiež povinný
  pre merge.

Jenkins je celý nastavený ako kód (Configuration as Code a Job DSL v
`pouw-deploy/jenkins/`), takže sa dá kedykoľvek znova postaviť a zmeny idú cez
pull request.

## Dôsledky

- CI je na dvoch miestach. Pravidlo: test jednej služby patrí do jej `ci-ok`,
  test spolupráce služieb do `e2e/` v `pouw-deploy`.
- Zmeny v UI Jenkinsu sa pri reštarte stratia, meniť treba súbory v
  `jenkins/casc/`.
- Jenkins musí byť dostupný z internetu (webhooky, prihlásenie cez GitHub),
  preto beží za HTTPS proxy, pozri [ADR-0005](0005-https-cez-caddy.md).
- Buildy bežia na našom serveri, preto sú izolované od stagingu, pozri
  [ADR-0002](0002-izolovany-docker-pre-buildy.md).
- Postup a podrobnosti: [CI/CD](../server/ci-cd.md).
