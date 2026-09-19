# Portfolio

Henkilökohtainen portfolio, julkaistaan GitHub Pagesin kautta.

## Rakenne

- `index.md` — etusivu, listaa projektit automaattisesti
- `_posts/` — projektikirjoitukset, tiedostonimi muotoa `VVVV-KK-PP-otsikko.md`
- `kuvat/` — kuvakaappaukset
- `_config.yml` — sivuston asetukset

## Uuden projektin lisääminen

Luo tiedosto `_posts/2026-01-15-projektin-nimi.md` ja aloita se front matterilla:

```yaml
---
layout: post
title: "Projektin otsikko"
excerpt: "Yhden virkkeen tiivistelmä, näkyy etusivun listauksessa."
---
```

Kuvat viitataan juuresta: `![Kuvateksti](/kuvat/tiedosto.png)`

Otsikkotasot alkavat postissa tasolta `##`, koska teema tulostaa otsikon
front matterista tasolla `#`.

## Julkaisuperiaatteet

Repo on julkinen ja koko commit-historia on julkinen. Kerran työnnettyä
salaisuutta ei saa pois pelkällä poistavalla commitilla.

Ennen jokaista committia:

- Ei salasanoja, API-avaimia eikä yksityisiä avaimia — ei tekstissä eikä kuvissa.
- Kuvakaappauksista peitetään asiakkaan tai työnantajan domainit, julkiset
  IP-osoitteet, sisäverkon osoitteet, käyttäjätunnukset ja sähköpostiosoitteet.
- Peittäminen tehdään kuvankäsittelyssä ja kuva tallennetaan uudelleen. PDF- tai
  dokumenttieditorissa kuvan päälle piirretty laatikko jättää alkuperäisen
  kuvan tiedostoon.
- Alkuperäisiä työdokumentteja ei julkaista sellaisenaan.
