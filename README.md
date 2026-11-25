# pouw-docs

## Štruktúra

```text
index.md        úvodná stránka webu
tim.md          tím a role
architecture/   architektúra systému
meetings/       zápisnice zo stretnutí
decisions/      architektonické rozhodnutia, ADR (šablóna TEMPLATE.md)
```

## Lokálne spustenie

Dokumentácia sa publikuje ako web cez GitHub Pages. Lokálny náhľad:

```bash
bundle install
bundle exec jekyll serve --livereload   # http://127.0.0.1:4000
```

Kontroly Markdownu a pravopisu:

```bash
npm install
npm run lint
npm run spell
```

Podrobnosti: [Ako písať dokumentáciu](ako-pisat.md).
