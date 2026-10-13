---
title: "ADR-0005: HTTPS cez Caddy"
parent: Rozhodnutia (ADR)
nav_order: 5
---

# ADR-0005: HTTPS cez Caddy a sslip.io, ostatné porty iba lokálne

**Stav:** prijaté
**Dátum:** 2026-10-12
**Autor:** @Faustynn

## Kontext

Jenkins a Grafana musia byť dostupné z internetu: prihlásenie cez GitHub
(OAuth vyžaduje HTTPS callback), webhooky z GitHubu a prístup tímu. Heslá a
cookies nesmú ísť po sieti nešifrované.

Na serveri FIIT je zvonka otvorený aj rozsah portov 8000-10000 a Docker
obchádza `ufw`, takže port publikovaný na `0.0.0.0` je hneď verejný. Vlastnú
doménu zatiaľ nemáme.

## Možnosti

Reverse proxy:

1. **Nginx + certbot**
   - výhody: rozšírené, veľa návodov;
   - nevýhody: viac konfigurácie, obnovu certifikátov treba riešiť zvlášť.
2. **Traefik**
   - výhody: konfigurácia cez labels v Docker Compose;
   - nevýhody: zložitejší model, konfigurácia roztrúsená po službách.
3. **Caddy**
   - výhody: certifikáty Let's Encrypt získava aj obnovuje sám; celá
     konfigurácia má pár riadkov;
   - nevýhody: menej rozšírený v tíme.

Doména:

1. **Vlastná doména alebo subdoména FIIT**: pekná adresa, ale treba ju vybaviť.
2. **sslip.io** (`jenkins.<ip-s-pomlčkami>.sslip.io`): funguje hneď, bez
   registrácie; závisí od externej služby.

## Rozhodnutie

Caddy (`proxy/Caddyfile`, profil `proxy`) je jediná služba s portami 80 a 443
na všetkých rozhraniach. Všetky ostatné služby majú porty iba na `127.0.0.1`.
Adresy sú zatiaľ na sslip.io.

## Dôsledky

- Zvonka sú dostupné iba `jenkins.147-175-151-18.sslip.io` a
  `grafana.147-175-151-18.sslip.io`, obe s platným certifikátom. Prometheus a
  ďalšie služby zvonka dostupné nie sú.
- Porty 80 a 443 musia byť otvorené z internetu, inak Caddy nezíska certifikát.
- Nová služba s webovým rozhraním potrebuje blok v `Caddyfile` a meno v `.env`.
- Prechod na vlastnú doménu znamená zmeniť `JENKINS_HOST`, `GRAFANA_HOST`,
  `JENKINS_URL` a `GRAFANA_URL` v `.env`, callback v OAuth App a URL webhooku v
  GitHub App.
- Ak by sslip.io prestalo fungovať, adresy prestanú fungovať tiež; riešenie je
  vlastná doména.
