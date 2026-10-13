---
title: Prevádzka
parent: Server
nav_order: 3
---

# Prevádzka

Bežná práca so staging serverom a repozitárom `pouw-deploy`. Príkazy na serveri
spúšťaj ako `ubuntu` v `~/pouw/pouw-deploy`.

## Stav a logy

```bash
make ps                     # kontajnery a ich zdravie
make logs s=jenkins         # logy jednej služby: jenkins, proxy, grafana, prometheus, ...
tail deploy-history.log     # nasadenia a automatické rollbacky
cat versions.env            # verzie služieb z posledného nasadenia
systemctl status pouw-secrets
```

Bez SSH prístupu: GitHub → Actions → **Deploy** → akcia `status`.

## Nasadenie a rollback

Na serveri nikdy neupravuj súbory ani nespúšťaj `git pull` ručne. Každá zmena
ide cez `main` a workflow Deploy, ktorý ju zapíše do `deploy-history.log`.

| Cieľ | Ako |
| --- | --- |
| Nasadiť posledný otestovaný stav | nič, urobí to večerný build, ak prejdú testy |
| Nasadiť `main` hneď | Actions → Deploy → akcia `deploy`, `sha` prázdne |
| Rollback | Actions → Deploy → akcia `deploy`, `sha` = starší commit z `main` (pozri `deploy-history.log`), `versions` = verzie z toho istého riadku, ak chceš vrátiť aj služby |
| Reštartovať všetko | Actions → Deploy → akcia `restart` (načíta aj heslá a konfiguráciu Jenkinsu) |
| Zistiť, čo beží | Actions → Deploy → akcia `status` |

`remote.sh` prijme iba celý SHA commitu, ktorý je v `origin/main`. Ak sa nová
verzia nenaštartuje (`make up` zlyhá, napríklad na healthchecku), vráti
predchádzajúci commit aj verzie a beh skončí chybou.

## Zmena konfigurácie Jenkinsu

| Zmena | Kde | Kedy začne platiť |
| --- | --- | --- |
| Pluginy | `jenkins/plugins.txt` | po nasadení (`make up` prestavia obraz) |
| Agent, credentials, e-mail, prihlásenie | `jenkins/casc/*.yaml` | po nasadení a Deploy → `restart`; CasC sa číta iba pri štarte Jenkinsu |
| Pre ktoré repozitáre sú joby | `jenkins/casc/jobs/github.groovy` | po nasadení a `restart` |
| Pipeline, čas večerného buildu, príjemca e-mailov | `Jenkinsfile` | pri ďalšom builde, Jenkins ho číta z gitu |
| Správcovia, adresy | `.env` na serveri | `make up` na serveri |

Zmeny urobené v UI Jenkinsu sa pri ďalšom reštarte prepíšu.

Zmenu Jenkinsu vyskúšaš pred merge lokálne: v `.env` nastav
`COMPOSE_PROFILES=jenkins` a `JENKINS_LOCAL_BRANCH=<tvoja vetva>`, potom
`make up`. Vetva musí byť pushnutá, Jenkins číta `Jenkinsfile` z GitHubu.
Lokálne sa nedá vyskúšať iba prihlásenie cez GitHub a priečinok organizácie.

## Heslá

Všetky heslá sú v `secrets.list`. Prečo práve `pass`, je v
[ADR-0003](../decisions/0003-hesla-v-pass.md).

| Druh | Kto ho vytvorí | Príklady |
| --- | --- | --- |
| `random` | `pass-init.sh` | admin Grafany, admin Jenkinsu, secret webhookov |
| `manual` | ty, `pass insert -m` | GitHub App, OAuth App, SMTP |
| `ssh-key` | `secrets.sh` pri každom štarte, nie je v `pass` | kľúč Jenkins → agent |

Kontajnery dostanú heslá ako súbory v `/run/secrets/<meno>`, nikdy cez
premenné prostredia. Na serveri sú súbory v `/run/pouw-secrets` (tmpfs,
po vypnutí zmiznú a pri štarte ich znova vytvorí `pouw-secrets.service`).

**Zmena hesla**

```bash
pass insert -m -f pouw/<cesta>    # alebo: pass generate -f --no-symbols pouw/<cesta> 32
make restart                      # prepíše /run/pouw-secrets a znova vytvorí kontajnery
```

Samotné `make up` nestačí: kontajner drží starý súbor, kým ho niekto znova
nevytvorí.

**Nové heslo**: riadok v `secrets.list` → položka v `secrets:` v `compose.yml`
(na najvyššej úrovni aj pri službe) → `./scripts/pass-init.sh` (vygeneruje
`random`) alebo `pass insert -m` (`manual`) → nasadenie.

**Záloha úložiska.** Úložisko je git repozitár zašifrovaný pre kľúč servera a
kľúče správcov. Skopíruj ho na bezpečné miesto, napríklad do súkromného
repozitára:

```bash
pass git remote add origin git@github.com:<súkromný-repozitár>.git
pass git push -u origin main
```

Obnova potrebuje kľúč, pre ktorý je úložisko zašifrované. Preto spusti
`pass-init.sh` aspoň s jedným GPG kľúčom správcu.

## Nový test siete

Testy celej siete sú v `e2e/smoke.sh`, jedna kontrola na riadok:

```bash
check "<popis>" <služba> '<shell príkaz, ktorý beží v kontajneri služby>'
```

Príkaz beží cez `docker compose exec`, takže nepotrebuje porty na hostiteľovi a
funguje lokálne, v Jenkinse aj na stagingu. Každá kontrola je jeden test v
záložke **Tests** v Jenkinse. Večerné nasadenie prebehne, iba ak prejdú všetky.

Príklady testov, ktoré dávajú zmysel, keď budú služby hotové:

```bash
check "node produces blocks" node-1 '...'
check "indexer sees the latest block" indexer '...'
check "workload submits a task and gets a proof" workload '...'
```

Lokálne ho vyskúšaš cez `make images test IMAGE_TAG=local` (postaví služby z
`main`), potom `make test-down`.

## Nová služba

Príklad: `pouw-node` dostane `Dockerfile`.

1. V repozitári služby: `Dockerfile`. `docker.yml` potom pri každom pushi do
   `main` publikuje `ghcr.io/pouw-zk/pouw-node:main` a `:sha-<7>`. Server
   sťahuje bez prihlásenia, takže balíček v GHCR musí byť verejný
   (organizácia → Packages → pouw-node → Package settings → visibility).
2. V `compose.yml`: odkomentuj alebo pridaj službu s
   `image: ghcr.io/pouw-zk/pouw-node:${NODE_VERSION:-main}` a healthcheckom.
   Porty na hostiteľovi iba ak sú naozaj potrebné; vtedy na `127.0.0.1` a v
   `compose.test.yml` ich vypni cez `ports: !reset []`.
3. Nové heslá: pozri časť Heslá vyššie.
4. Metriky: scrape job v `monitoring/prometheus/prometheus.yml`.
5. Testy: kontroly v `e2e/smoke.sh`.

`scripts/build-images.sh`, `Makefile` (`IMAGE_TAG`) a `remote.sh` už poznajú
`pouw-node`, `pouw-indexer`, `pouw-workload` a `pouw-dashboard`. Službu s novým
menom treba pridať do všetkých troch.

## Kapacita

Agent má 2 executory (`numExecutors` v `jenkins/casc/jenkins.yaml`), takže
naraz bežia najviac dva buildy, každý s celou testovacou kópiou siete.
Orientačne podľa pamäte servera: 8 GB a menej → 1, 16 GB → 2, 32 GB a viac → 3
až 4. Zníž, ak buildy spomaľujú staging; zvýš, ak buildy čakajú vo fronte.

Izolovaný Docker drží cache v zväzku `jenkins-docker-data`, nepoužívané obrazy
sa mažú po každom builde. Ak sa disk aj tak zaplní:

```bash
docker compose exec jenkins-docker docker system prune -af
```

## Bezpečnosť

- **Zvonka dostupné:** iba proxy na 80/443 (HTTPS, Let's Encrypt). Všetky
  ostatné porty sú na `127.0.0.1`.
- **Prístup do Jenkinsu:** prihlásenie cez GitHub, členovia PoUW-ZK môžu joby
  pozerať a spúšťať, `JENKINS_ADMINS` spravujú Jenkins. Anonym vidí iba stav
  buildu (odznak v README).
- **Buildy:** bežia na agentovi ako používateľ `jenkins` proti izolovanému
  Dockeru. Nevidia kontajnery stagingu ani `/run/pouw-secrets`. PR z forkov sa
  nestavajú.
- **Kľúč na nasadenie:** forced command (`remote.sh`) v `authorized_keys`.
  Uniknutý kľúč vie iba nasadiť commit, ktorý už je v `main`, reštartovať a
  ukázať stav.
- **Heslá:** zašifrované v `pass`, dešifrované iba do tmpfs, do kontajnerov ako
  súbory. GitHub má iba kľúč na nasadenie, kľúč GitHub App je v `pass` na serveri.
- **Skenovanie:** Trivy a actionlint pri každom PR (workflow `security`).
  `jenkins/agent.Dockerfile` zámerne končí `USER root` (potrebuje ho sshd,
  buildy bežia ako `jenkins`), preto má `# trivy:ignore:DS-0002`. Tento riadok
  nemaž.

## Riešenie problémov

| Príznak | Príčina | Riešenie |
| --- | --- | --- |
| `make up`: `Missing in pass: pouw/smtp/...` | SMTP nie je nastavené | v poriadku, ak je to zámer; pozri časť SMTP v [inštalácii](instalacia.md) |
| Jenkins nenabehne, v `make logs s=jenkins` je `ConfiguratorException` | chybná hodnota v `jenkins/casc` alebo v hesle | oprav YAML alebo heslo; `pass show pouw/github/app-key \| head -1` musí byť `-----BEGIN PRIVATE KEY-----` |
| Prihlásenie cez GitHub sa zacyklí alebo zlyhá | callback URL v OAuth App nesedí s `JENKINS_URL` | callback musí byť `https://<host>/securityRealm/finishLogin` |
| Priečinok PoUW-ZK je prázdny | GitHub App nie je nainštalovaná alebo nemá práva | Install App → All repositories; priečinok → Scan Organization Now |
| PR nespúšťajú buildy | zlá URL alebo secret webhooku | GitHub App → Advanced → Recent Deliveries ukáže chybu |
| Bez certifikátu, prehliadač hlási chybu TLS | zatvorené porty 80/443 alebo DNS neukazuje na server | `make logs s=proxy`, zvonka `nc -vz <ip> 443` |
| Večerné nasadenie: `Deploy workflow run not found` | GitHub App nemá Actions: Read and write | nastavenia aplikácie → Permissions, potom potvrď zmenu v inštalácii |
| Beh Deploy zlyhá na SSH | zlé hodnoty `DEPLOY_*`, chýba riadok v `authorized_keys` alebo je port 22 zvonka zatvorený | pozri časť 7 v [inštalácii](instalacia.md) |
| Beh Deploy: `Commit ... is not in main` | `sha` je z inej vetvy | použi commit z `main` |
| Workflow Security padá na DS-0002 | z `agent.Dockerfile` zmizol riadok `trivy:ignore` | vráť ho nad `USER root` |
| `pass` hlási `Please tell me who you are` | stará verzia `pass-init.sh` bez git identity | `pass git config user.name "PoUW server"`, `pass git config user.email pouw@$(hostname)`, `pass git add -A && pass git commit -m "Add secrets"` |
| Po reštarte servera kontajnerom chýbajú heslá | zlyhala `pouw-secrets.service` | `systemctl status pouw-secrets`, `journalctl -u pouw-secrets` |
| Lokálny Jenkins: `No rule to make target 'check'` | job vytvorený v UI s vloženým skriptom nemá checkout | používaj job `pouw-deploy` z CasC, job z UI zmaž |
| Lokálny Jenkins sa nedostane na GitHub | pokazená sieť Docker Desktop | reštartuj Docker Desktop; `docker run --rm curlimages/curl -sI https://github.com` |
| `make test`: `port is already allocated` | nová služba publikuje port na hostiteľovi | pridaj ju do `compose.test.yml` s `ports: !reset []` |
