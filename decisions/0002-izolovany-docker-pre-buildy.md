---
title: "ADR-0002: Izolovaný Docker pre buildy"
parent: Rozhodnutia (ADR)
nav_order: 2
---

# ADR-0002: Buildy Jenkinsu v izolovanom Dockeri

**Stav:** prijaté
**Dátum:** 2026-10-12
**Autor:** @Faustynn

## Kontext

Build v Jenkinse spúšťa Docker: stavia obrazy služieb a spúšťa testovaciu
kópiu siete. Na tom istom serveri beží staging s heslami v `/run/pouw-secrets`.

Kto má prístup k Dockeru servera, má v praxi root na serveri. `Jenkinsfile`
pritom môže zmeniť ktokoľvek v tíme v pull requeste, a ten sa postaví ešte pred
schválením. Build teda nesmie dosiahnuť na staging, jeho dáta ani heslá.

## Možnosti

1. **Buildy priamo na Jenkinse s `docker.sock` servera**
   - výhody: najjednoduchšie, rýchla cache;
   - nevýhody: každý PR má root na serveri, vidí heslá aj kontajnery stagingu.
2. **Samostatný agent s `docker.sock` servera**
   - výhody: Jenkins controller je oddelený;
   - nevýhody: z pohľadu bezpečnosti to isté ako možnosť 1.
3. **Samostatný agent s vlastným Dockerom (Docker-in-Docker)**
   - výhody: build vidí iba vlastný Docker, nie server; staging, heslá a
     Jenkins sú mimo jeho dosahu;
   - nevýhody: kontajner `jenkins-docker` beží privileged, má vlastnú cache
     obrazov a zaberá disk navyše.
4. **Kubernetes agenti alebo samostatný build server**
   - výhody: najlepšia izolácia, škálovanie;
   - nevýhody: na jeden server zbytočne zložité, ďalší stroj nemáme.

## Rozhodnutie

Možnosť 3. Jenkins controller má 0 executorov a nič nestavia. Všetky buildy
bežia na agentovi `jenkins-agent` (SSH agent), ktorý posiela Docker príkazy do
`jenkins-docker` cez TLS. Agent a `jenkins-docker` sú v samostatnej Docker
sieti, nie v sieti so službami stagingu.

## Dôsledky

- Testovacia kópia siete beží v `jenkins-docker`, nie v Dockeri servera. Nemá
  porty na hostiteľovi (`compose.test.yml`), takže sa nebije so stagingom ani
  s druhým súbežným buildom.
- Cesta k workspace musí byť v agentovi aj v `jenkins-docker` rovnaká (zdieľaný
  zväzok `/home/jenkins`), inak by nefungovali bind mounty z workspace.
- Obraz agenta končí `USER root`, lebo sshd potrebuje root. Buildy bežia ako
  používateľ `jenkins`. Trivy to hlási ako DS-0002, preto má `agent.Dockerfile`
  výnimku `trivy:ignore:DS-0002`.
- Cache obrazov v `jenkins-docker` rastie, po každom builde sa mažú nepoužívané
  obrazy.
- Pull requesty z forkov sa nestavajú vôbec, ani izolácia nie je dôvod spúšťať
  cudzí kód na našom serveri.
