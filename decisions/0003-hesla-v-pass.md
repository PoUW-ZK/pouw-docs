---
title: "ADR-0003: Heslá v pass"
parent: Rozhodnutia (ADR)
nav_order: 3
---

# ADR-0003: Heslá v pass, do kontajnerov ako súbory

**Stav:** prijaté
**Dátum:** 2026-10-12
**Autor:** @Faustynn

## Kontext

Služby potrebujú heslá a kľúče: admin Grafany, kľúč GitHub App, OAuth secret,
secret webhookov, neskôr heslo databázy. Repozitáre sú verejné, takže heslá
nesmú byť v gite. Server sa musí vedieť po reštarte rozbehnúť bez človeka a
stratený server nesmie znamenať stratené heslá.

## Možnosti

1. **Súbor `.env` na serveri**
   - výhody: najjednoduchšie;
   - nevýhody: heslá nezašifrované na disku, bez histórie a zálohy; v
     premenných prostredia ich vidí `docker inspect` a môžu uniknúť do logov.
2. **GitHub Secrets**
   - výhody: už ich používame pre Actions;
   - nevýhody: dostupné iba počas behu workflow, server ich nevidí; každé
     nasadenie by ich muselo prenášať.
3. **HashiCorp Vault (alebo OpenBao)**
   - výhody: dynamické heslá, audit, rotácia;
   - nevýhody: ďalšia služba, ktorú treba po každom reštarte odomknúť (unseal),
     zálohovať a aktualizovať; keď nebeží, nenabehne nič. Na jeden server a
     desať hesiel prehnané. Jenkins plugin pre Vault mal navyše bezpečnostné
     upozornenie.
4. **`pass` (GPG) na serveri, dešifrovanie do tmpfs, Docker secrets**
   - výhody: heslá zašifrované, úložisko je git s históriou a dá sa zálohovať;
     správcovia ho vedia dešifrovať aj bez servera; žiadna ďalšia služba;
   - nevýhody: kľúč servera je bez passphrase (chráni ho iba prístup k účtu);
     ručné heslá treba vložiť pri inštalácii.
5. **SOPS + age (zašifrované heslá priamo v gite)**
   - výhody: heslá verzionované spolu s kódom;
   - nevýhody: ďalší nástroj a správa kľúčov pre celý tím; pri jednom serveri
     bez výhody oproti `pass`.

## Rozhodnutie

Možnosť 4:

- `secrets.list` v `pouw-deploy` je jediný zoznam všetkých hesiel: meno súboru,
  cesta v `pass` a druh (`random`, `manual`, `ssh-key`).
- `pass-init.sh` vygeneruje náhodné heslá a zašifruje úložisko pre kľúč servera
  a kľúče správcov.
- `secrets.sh` pri každom štarte (systemd `pouw-secrets.service`, ešte pred
  Dockerom) dešifruje heslá do `/run/pouw-secrets` na tmpfs.
- Kontajnery ich dostanú ako súbory v `/run/secrets/`, nikdy cez premenné
  prostredia.
- Lokálne sa namiesto `pass` vygenerujú náhodné heslá do `.secrets/`.

## Dôsledky

- Na disku servera nie je žiadne heslo v čitateľnej podobe, po vypnutí z
  tmpfs zmiznú.
- Po zmene hesla treba `make restart`, kontajner inak drží starý súbor.
- Úložisko treba zálohovať (`pass git push` do súkromného repozitára) a
  zašifrovať aj pre aspoň jedného správcu.
- Ak bude serverov viac alebo prejdeme na Kubernetes, prirodzený ďalší krok je
  SOPS alebo Vault. Prechod uľahčí, že všetky heslá sú opísané v `secrets.list`.
