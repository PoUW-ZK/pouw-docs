---
title: Inštalácia servera
parent: Server
nav_order: 2
---

# Inštalácia servera

Inštalácia staging servera od nuly a jeho prepojenie s GitHubom. Potrebuješ
SSH prístup so `sudo` na server a práva vlastníka v organizácii PoUW-ZK.

Aktuálny server je `pouw-dev` (147.175.151.18, používateľ `ubuntu`, repozitár
v `~/pouw/pouw-deploy`). Pri inštalácii iného servera tieto hodnoty nahraď.

## 1. Pred začiatkom

**Adresa.** Bez vlastnej domény použijeme [sslip.io](https://sslip.io): meno
`jenkins.147-175-151-18.sslip.io` sa preloží na `147.175.151.18`. Ďalej
`<host>` znamená adresu Jenkinsu. Prečo, je v
[ADR-0005](../decisions/0005-https-cez-caddy.md).

Na serveri over:

```bash
curl -4 -s ifconfig.me; echo     # verejná IP
ip -4 -br addr                   # musí obsahovať tú istú IP, inak je server za NAT
```

**Porty.** 80 a 443 musia byť dostupné z internetu (Let's Encrypt a webhooky
z GitHubu). Port 22 je na sieti FIIT zvonka zatvorený a nevadí to: nasadenie
beží na self-hosted runneri priamo na serveri (časť 7). Z počítača mimo servera:

```bash
nc -vz 147.175.151.18 443   # "Connection refused" je pred spustením v poriadku, timeout nie
```

Na sieti FIIT je zvonka otvorený aj rozsah 8000-10000 a Docker obchádza `ufw`.
Preto `compose.yml` viaže všetky porty okrem proxy iba na `127.0.0.1`.

## 2. Príprava servera

```bash
git clone https://github.com/PoUW-ZK/pouw-deploy.git ~/pouw/pouw-deploy
cd ~/pouw/pouw-deploy
sudo ./scripts/server-setup.sh
exit    # znova sa prihlás, aby platila skupina docker
```

`server-setup.sh` nainštaluje Docker, `pass`, `gnupg`, `make`, `git` a
`openssl`. Vytvorí tmpfs adresár `/run/pouw-secrets` a systemd službu
`pouw-secrets.service`, ktorá pri každom štarte servera dešifruje heslá ešte
pred spustením Dockeru.

Potom vytvor úložisko hesiel:

```bash
cd ~/pouw/pouw-deploy
./scripts/pass-init.sh [GPG_ID_SPRÁVCU ...]
```

`pass-init.sh`:

1. vytvorí GPG kľúč servera bez passphrase, aby nasadenie a reštart nepotrebovali človeka;
2. inicializuje `pass`, zašifrovaný pre kľúč servera a pre kľúče správcov
   z argumentov (ich verejné kľúče najprv importuj: `gpg --import spravca.asc`);
3. vygeneruje všetky heslá druhu `random` zo `secrets.list`;
4. na konci vypíše heslá druhu `manual`, ktoré treba vložiť ručne. Odkiaľ ich
   vziať, popisujú časti 3 až 5.

Aspoň jeden kľúč správcu veľmi odporúčame. Bez neho vie úložisko dešifrovať
iba server a stratený server znamená stratené heslá.

## 3. GitHub OAuth App (prihlásenie do Jenkinsu)

Cez túto aplikáciu sa používatelia prihlasujú do Jenkinsu GitHub účtom.

1. Otvor <https://github.com/organizations/PoUW-ZK/settings/applications> → **New OAuth App**.
2. Vyplň:
   - Application name: `PoUW-ZK Jenkins`
   - Homepage URL: `https://<host>/`
   - Authorization callback URL: `https://<host>/securityRealm/finishLogin`
3. **Register application**, potom **Generate a new client secret** (zobrazí sa iba raz).
4. Na serveri:

   ```bash
   pass insert -m pouw/github/oauth-client-id       # vlož, Enter, Ctrl+D
   pass insert -m pouw/github/oauth-client-secret
   ```

## 4. GitHub App (prístup Jenkinsu k API)

Cez túto aplikáciu Jenkins číta repozitáre, dostáva webhooky, hlási výsledky
do PR a spúšťa workflow Deploy.

1. Otvor <https://github.com/organizations/PoUW-ZK/settings/apps/new>.
2. Vyplň:

   | Pole | Hodnota |
   | --- | --- |
   | GitHub App name | `pouw-zk-jenkins` (musí byť unikátne na celom GitHube) |
   | Homepage URL | `https://<host>/` |
   | Callback URL, Setup URL | prázdne |
   | Request user authorization, Device Flow | vypnuté |
   | Webhook → Active | zapnuté |
   | Webhook URL | `https://<host>/github-webhook/` |
   | Webhook secret | výstup `pass show pouw/github/webhook-secret` na serveri |
   | Where can this app be installed | Only on this account |

3. Oprávnenia:

   | Repository permission | Prístup | Na čo |
   | --- | --- | --- |
   | Actions | Read and write | spustenie workflow Deploy po večernom builde |
   | Checks | Read and write | výsledky buildov v PR |
   | Commit statuses | Read and write | statusy `continuous-integration/jenkins/*` |
   | Contents | Read-only | klonovanie repozitárov |
   | Metadata | Read-only | vyžaduje GitHub |
   | Pull requests | Read-only | hľadanie PR |

   | Organization permission | Prístup | Na čo |
   | --- | --- | --- |
   | Members | Read-only | členstvo v organizácii pri skenovaní |

4. Subscribe to events: **Push**, **Pull request**, **Repository**.
5. **Create GitHub App**. Na stránke aplikácie:
   - **App ID** hore (číslo, napríklad `5292663`). Client ID vedľa neho nepoužívame.
   - Úplne dole **Private keys** → **Generate a private key**. Nie
     „Generate a new client secret“, to je niečo iné.
6. Vľavo **Install App** → **Install** pri PoUW-ZK → **All repositories**.
7. Ulož App ID a kľúč. Jenkins potrebuje kľúč vo formáte PKCS#8, GitHub dáva
   PKCS#1 (`BEGIN RSA PRIVATE KEY`). Skonvertuj ho na notebooku a pošli rovno
   do `pass` na serveri, aby nikde na serveri neležal nezašifrovaný:

   ```bash
   printf '5292663' | ssh ubuntu@147.175.151.18 'pass insert -m -f pouw/github/app-id'
   openssl pkcs8 -topk8 -nocrypt -in ~/Downloads/pouw-zk-jenkins.*.private-key.pem \
     | ssh ubuntu@147.175.151.18 'pass insert -m -f pouw/github/app-key'
   ssh ubuntu@147.175.151.18 'pass show pouw/github/app-key | head -1'   # -----BEGIN PRIVATE KEY-----
   rm ~/Downloads/pouw-zk-jenkins.*.private-key.pem
   ```

   Stratený kľúč nevadí: na tej istej stránke vygeneruj nový a ulož ho znova.

## 5. SMTP (nepovinné)

Jenkins posiela e-maily o zlyhanom večernom builde cez SMTP účet. Túto časť
môžeš preskočiť, všetko ostatné funguje aj bez nej. `make up` iba upozorní, že
chýba `pouw/smtp/*`.

Odporúčame samostatný Gmail účet projektu (nie osobný) so zapnutým dvojfázovým
overením a [heslom aplikácie](https://myaccount.google.com/apppasswords):

```bash
pass insert -m pouw/smtp/username    # napr. pouw.zk.jenkins@gmail.com
pass insert -m pouw/smtp/password    # 16-znakové heslo aplikácie, bez medzier
```

a v `.env`:

```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_FROM=pouw.zk.jenkins@gmail.com   # musí to byť ten istý účet
```

Over, že sa server k nemu dostane: `nc -vz smtp.gmail.com 587`. Osobné
univerzitné heslo sem nevkladaj.

## 6. Konfigurácia a spustenie

```bash
cd ~/pouw/pouw-deploy
cat > .env <<'EOF'
COMPOSE_PROFILES=jenkins,proxy,host-metrics
SECRETS_SOURCE=pass
SECRETS_DIR=/run/pouw-secrets

JENKINS_HOST=jenkins.147-175-151-18.sslip.io
GRAFANA_HOST=grafana.147-175-151-18.sslip.io
JENKINS_URL=https://jenkins.147-175-151-18.sslip.io/
GRAFANA_URL=https://grafana.147-175-151-18.sslip.io/
ACME_EMAIL=xmeredov@stuba.sk

JENKINS_AUTH=github
JENKINS_ADMINS=Faustynn
EOF

make up      # prvýkrát sa stavia obraz Jenkinsu, 5 až 10 minút
make ps      # všetko Up, jenkins/grafana/prometheus (healthy)
```

| Premenná | Význam |
| --- | --- |
| `COMPOSE_PROFILES` | na serveri `jenkins,proxy,host-metrics` (`host-metrics` = node-exporter a cAdvisor); bez profilu beží iba databáza a monitoring; Hasura a služby siete pribudnú s profilom `network` |
| `SECRETS_SOURCE`, `SECRETS_DIR` | heslá z `pass`, dešifrované do tmpfs |
| `JENKINS_HOST`, `GRAFANA_HOST` | mená, na ktoré proxy odpovedá a pre ktoré získa certifikát |
| `JENKINS_URL`, `GRAFANA_URL` | verejné adresy; lomka na konci je povinná |
| `ACME_EMAIL` | účet Let's Encrypt, upozornenia na expirujúce certifikáty |
| `JENKINS_AUTH` | na serveri `github`, na notebooku `local` (predvolené) |
| `JENKINS_ADMINS` | GitHub účty s plnými právami v Jenkinse, oddelené čiarkou; ostatní členovia organizácie môžu joby pozerať a spúšťať |

`.env` nie je v gite. Neobsahuje heslá, iba nastavenia servera.

## 7. Nasadenie z GitHubu

Port 22 je zvonka zatvorený, takže GitHub sa na server pripojiť nevie. Workflow
Deploy preto beží na self-hosted runneri priamo na serveri: runner sa sám
pripája na GitHub cez HTTPS a preberá úlohy. Úloha sa potom cez SSH pripojí na
`127.0.0.1` kľúčom, ktorý smie spustiť iba `scripts/remote.sh`.

1. Runner. Na notebooku získaj registračný token (platí hodinu):

   ```bash
   gh api -X POST repos/PoUW-ZK/pouw-deploy/actions/runners/registration-token --jq .token
   ```

   a na serveri ho nainštaluj:

   ```bash
   cd ~/pouw/pouw-deploy
   sudo ./scripts/runner-setup.sh <token>
   ```

   Runner beží ako systémový používateľ `gh-runner` (bez `sudo`, mimo skupiny
   `docker`) so štítkom `staging`. Na GitHube ho uvidíš v pouw-deploy →
   Settings → Actions → Runners ako **Idle**.

2. Na notebooku vytvor kľúč iba na nasadenie:

   ```bash
   ssh-keygen -t ed25519 -N '' -C github-actions -f deploy_key
   ```

3. Na serveri pridaj do `~/.ssh/authorized_keys` jeden riadok (cesta musí presne sedieť):

   ```text
   command="/home/ubuntu/pouw/pouw-deploy/scripts/remote.sh",restrict ssh-ed25519 AAAA... github-actions
   ```

4. Na GitHube: pouw-deploy → Settings → Environments → **New environment** `staging`.
   Nepridávaj required reviewers, inak by večerné nasadenie čakalo na človeka.

   | Druh | Meno | Hodnota |
   | --- | --- | --- |
   | Secret | `DEPLOY_SSH_KEY` | obsah súboru `deploy_key` (súkromný kľúč) |
   | Variable | `DEPLOY_HOST` | `127.0.0.1` |
   | Variable | `DEPLOY_USER` | `ubuntu` |
   | Variable | `DEPLOY_KNOWN_HOSTS` | výstup `echo "127.0.0.1 $(cut -d' ' -f1,2 /etc/ssh/ssh_host_ed25519_key.pub)"` na serveri |

5. Zmaž `deploy_key` a `deploy_key.pub` z notebooku.
6. Vyskúšaj: GitHub → Actions → **Deploy** → Run workflow → akcia `status`.
   V súhrne behu je nasadený commit a zoznam kontajnerov.

## 8. Záverečné kontroly

1. <https://jenkins.147-175-151-18.sslip.io>: prihlás sa cez GitHub. V
   priečinku **PoUW-ZK** je `pouw-deploy` s vetvou `main`. Ak je prázdny,
   otvor priečinok → **Scan Organization Now**.
2. Prvý build `main` skončí zelený. Odvtedy je zaregistrovaný večerný časovač.
3. Runner je **Idle** a Actions → Deploy → `status` funguje (časť 7).
4. Na notebooku ako vlastník organizácie spusti `./setup-pouw-deploy.sh` z
   koreňa pracovného adresára PoUW-ZK. Kontrola z Jenkinsu sa tým stane
   povinnou pre PR v `pouw-deploy`.
5. Test reštartu (nepovinné, ale raz sa oplatí):

   ```bash
   sudo reboot
   # po nabehnutí:
   systemctl status pouw-secrets   # active (exited)
   cd ~/pouw/pouw-deploy && make ps
   ```

## Aktualizácia servera

Zmeny sa na server dostanú iba cez `main` a workflow Deploy (večer alebo
ručne), nikdy úpravou súborov priamo na serveri. Pozri [Prevádzka](prevadzka.md).
