---
title: "ADR-0004: Večerné nasadenie na staging"
parent: Rozhodnutia (ADR)
nav_order: 4
---

# ADR-0004: Večerné nasadenie na staging iba po úspešných testoch

**Stav:** prijaté
**Dátum:** 2026-10-12
**Autor:** @Faustynn

## Kontext

Staging má ukazovať aktuálny stav `main` všetkých služieb, aby ho tím mohol
skúšať a ukazovať. Služby sa ale menia v rôznych repozitároch nezávisle, takže
aj keď každá prejde vlastným `ci-ok`, ich kombinácia môže byť nefunkčná.
Staging sa nesmie rozbiť a nasadenie musí byť dohľadateľné a vratné.

## Možnosti

Kedy nasadzovať:

1. **Po každom merge do `main`** v ktoromkoľvek repozitári
   - výhody: staging je vždy najnovší;
   - nevýhody: nasadí sa aj kombinácia služieb, ktorú nikto spolu netestoval;
     veľa nasadení počas dňa.
2. **Iba ručne**
   - výhody: plná kontrola;
   - nevýhody: staging zastaráva, nikto si nespomenie.
3. **Každý večer po teste celej siete**
   - výhody: na staging ide iba to, čo spolu prešlo testami; raz denne, mimo
     pracovného času;
   - nevýhody: staging je najviac jeden deň pozadu.

Ako nasadzovať:

1. **Jenkins sa pripojí na server cez SSH priamo**
   - nevýhody: ďalší kľúč s prístupom na server, nasadenie bez záznamu na GitHube.
2. **Jenkins spustí workflow Deploy v GitHub Actions**
   - výhody: jedna cesta pre automatické aj ručné nasadenie; história v
     Actions; kľúč na server má iba environment `staging` na GitHube.

## Rozhodnutie

Večer (okolo 21:00) Jenkins postaví všetky služby z `main` od nuly, spustí
testovaciu kópiu siete a pustí testy. Iba ak prejdú, spustí cez GitHub App
workflow Deploy s commitom `pouw-deploy` a presnými verziami služieb
(`NODE_VERSION=sha-xxxxxxx`, …) a počká na výsledok.

Workflow sa pripojí na server kľúčom, ktorý smie spustiť iba
`scripts/remote.sh` (forced command v `authorized_keys`). `remote.sh`:

- prijme iba commit, ktorý je v `origin/main`, a verzie v presnom formáte;
- nasadí ho a verzie uloží do `versions.env`;
- ak sa nová verzia nenaštartuje, vráti predchádzajúci commit aj verzie;
- každé nasadenie a rollback zapíše do `deploy-history.log`.

Ručné nasadenie, rollback, reštart a stav idú cez ten istý workflow.

## Dôsledky

- Ak testy alebo nasadenie zlyhajú, staging zostane na predchádzajúcej verzii
  a príde e-mail.
- Na stagingu bežia presne otestované verzie. Obrazy `:sha-<7>` musí
  publikovať `docker.yml` v repozitári služby, inak nasadenie zlyhá a vráti sa.
- GitHub App potrebuje oprávnenie Actions: Read and write.
- Server musí byť dostupný cez SSH z GitHub Actions.
- Uniknutý kľúč na nasadenie vie iba nasadiť commit, ktorý už je v `main`,
  reštartovať a ukázať stav.
- Workflow Deploy nevracia ID behu, Jenkins ho hľadá podľa času spustenia. Keby
  niekto v tej istej minúte spustil Deploy ručne, Jenkins môže sledovať jeho beh.
- Postup a podrobnosti: [CI/CD](../server/ci-cd.md).
