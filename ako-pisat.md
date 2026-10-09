---
title: Ako písať dokumentáciu
nav_order: 8
---

# Ako písať dokumentáciu

Dokumentácia je obyčajný Markdown v repozitári `pouw-docs`. Po merge do `main`
sa web automaticky zostaví a publikuje.

## Nová stránka

Každá stránka začína hlavičkou (front matter):

```yaml
---
title: Názov v navigácii
parent: Zápisnice      # názov nadradenej sekcie, ak je to podstránka
nav_order: 2           # poradie v navigácii (nepovinné)
---
```

- Odkazy na iné stránky píš na `.md` súbory, napríklad `[ADR](../decisions/0001-kniznice.md)`.
  Pri zostavení sa automaticky zmenia na `.html`.
- Diagramy kresli priamo v Markdowne v bloku `mermaid`.
- Obrázky ukladaj vedľa stránky, súbory väčšie ako 5 MB patria do GitHub Releases.

## Lokálny náhľad

Potrebuješ Ruby 3+ a Bundler:

```bash
bundle install
bundle exec jekyll serve --livereload
```

Web beží na <http://127.0.0.1:4000>.

## Kontroly pred pull requestom

Rovnaké kontroly beží aj CI:

```bash
npm install
npm run lint     # formát Markdownu
npm run spell    # pravopis (slovenčina a angličtina)
```

Ak kontrola pravopisu hlási odborný výraz (napríklad názov knižnice),
pridaj ho do súboru `.cspell/project-words.txt`, jedno slovo na riadok.
