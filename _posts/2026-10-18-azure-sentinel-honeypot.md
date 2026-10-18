---
layout: post
title: "Azure Sentinel -honeypot: oikean brute force -hyökkäyksen tunnistus omilla säännöillä"
date: 2026-10-18
excerpt: "Internetiin avattu Windows Server Azuressa ja Microsoft Sentinel: kaksi omaa tunnistussääntöä (MITRE T1110), kontrolloidut testit ja oikea 81 yrityksen hyökkäys tunnistettuna, tutkittuna ja suljettuna. Säännöt julkaistaan koodina GitHub Actionsilla."
tags: [azure, microsoft-sentinel, kql, log-analytics, windows-server, azure-cli, github-actions, oidc, mitre-attack, soc]
---

**Lyhyesti:** Rakensin Azureen tarkoituksella internetiin avoimen Windows Server -koneen (honeypot = houkutuslaite) ja kytkin sen Microsoft Sentineliin. Kirjoitin kaksi omaa tunnistussääntöä RDP-brute forcelle, validoin molemmat omilla testeillä ja tunnistin ensimmäisen oikean hyökkäyksen vuorokauden sisällä. Säännöt ovat Gitissä, ja GitHub Actions julkaisee ne Sentineliin automaattisesti ilman tallennettuja salasanoja. Koko labra rakentuu ja purkautuu skripteillä. Toteutin projektin itsenäisesti lokakuussa 2026.

**Teknologiat:** Microsoft Azure · Microsoft Sentinel · Log Analytics · KQL · Azure Monitor Agent · Data Collection Rule · Windows Server 2025 (Trusted Launch) · Azure CLI · Bash · GitHub Actions · OIDC · MITRE ATT&CK

**Koodi:** [github.com/Tuomas404/azure-sentinel-honeypot](https://github.com/Tuomas404/azure-sentinel-honeypot)

---

## Tavoite

- Harjoitella SOC-työtä (tietoturvavalvomo) ja tunnistussääntöjen tekemistä oikealla hyökkäysdatalla
- Omat säännöt sekä nopealle että hitaalle (low-and-slow) salasanojen arvailulle
- Jokainen sääntö todistetusti toimiva: testattu ennen kuin siihen luotetaan
- Labra toistettavissa tyhjästä ja kulut hallinnassa

## Arkkitehtuuri

```
[Internet]
   │  RDP 3389
   ▼
[Windows Server 2025 VM]  Security-loki
   │  Azure Monitor Agent + Data Collection Rule
   ▼
[Log Analytics]
   │
   ▼
[Microsoft Sentinel]  omat analytiikkasäännöt (T1110)
   │
   ▼
[Incidentit]  tutkinta ja luokittelu
```

| Komponentti | Rooli |
|---|---|
| Windows Server 2025 VM | Houkutuslaite, RDP avoinna internetiin |
| Azure Monitor Agent + DCR | Kerää Windowsin Security-lokin ja lähettää sen Log Analyticsiin |
| Log Analytics | Lokien tallennus ja KQL-kyselyt |
| Microsoft Sentinel | Tunnistussäännöt, incidentit ja tutkinta |
| `deploy.sh` / `teardown.sh` | Koko labran rakennus ja purku Azure CLI:llä |

## Tunnistussäännöt

Molemmat säännöt tunnistavat lähde-IP:n entiteetiksi ja ryhmittelevät saman IP:n hälytykset yhdeksi incidentiksi.

| | Sääntö 1: nopea brute force | Sääntö 2: low-and-slow |
|---|---|---|
| Logiikka | ≥ 10 epäonnistunutta kirjautumista (4625) yhdestä IP:stä tunnissa | ≥ 20 / 24 h, vähintään 3 eri tuntina, alle 10 missään tunnissa |
| Ajo | Tunnin välein | 6 tunnin välein, 24 h ikkuna |
| MITRE ATT&CK | T1110 Brute Force | T1110 Brute Force |

Sääntö 2 on tehty hyökkääjälle, joka pysyttelee säännön 1 rajan alla. Se rajaa nopeat hyökkäykset pois, jotta sama toiminta ei tee kahta incidenttiä.

```kql
// Sääntö 2
SecurityEvent
| where EventID == 4625 and IpAddress != "-"
| summarize HourlyAttempts = count() by IpAddress, Hour = bin(TimeGenerated, 1h)
| summarize Attempts = sum(HourlyAttempts), ActiveHours = dcount(Hour),
            MaxPerHour = max(HourlyAttempts) by IpAddress
| where Attempts >= 20 and ActiveHours >= 3 and MaxPerHour < 10
```

## Validointi

**Testi 1, nopea brute force (17.10.2026):** 10 epäonnistunutta RDP-kirjautumista omasta IP:stä tunnin sisällä. Lokeissa 12 × EventID 4625 ja 1 × EventID 4740 (tili lukittui 10 yrityksen jälkeen). Sääntö 1 loi incidentin, joka suljettiin luokalla *Benign Positive* (oikea ilmiö, mutta ei uhka).

![Sääntö 1: incident auki](https://github.com/user-attachments/assets/fd066b66-af2d-45b8-a49e-0bc1dc616f7c)

**Testi 2, low-and-slow (18.10.2026):** 3 × 7 epäonnistunutta kirjautumista eri tunteina. Sääntö 2 loi incidentin, ja sääntö 1 pysyi oikein hiljaa. Suljettu luokalla *Benign Positive*.

![Sääntö 2: incident auki](https://github.com/user-attachments/assets/f71a3e9f-1109-4cfc-a407-dc0c337fb4c3)

## Ensimmäinen oikea hyökkäys (17.10.2026)

| | |
|---|---|
| Toiminta | 81 epäonnistunutta RDP-kirjautumista yhdestä IP:stä klo 15:59–19:58 UTC, enimmillään 21 tunnissa |
| IP:n sijainti | Intia (sijainti kertoo, missä IP on, ei kuka hyökkääjä on) |
| Kokeillut tunnukset | `Administrator`, `Admin`: kumpaakaan ei ole koneella |
| Onnistuneet etäkirjautumiset | Ei yhtään (ei EventID 4624 tyyppiä 3 tai 10 ulkoisista IP:istä 3 vuorokauteen) |
| Lopputulos | Ei murtoa. Toiminta loppui VM:n iltaiseen automaattiseen sammutukseen |
| Luokitus | *True Positive* (oikea uhka tunnistettu) |

Sääntö 1 loi yhdestä hyökkääjästä neljä incidenttiä, yhden jokaisella tunnin ajolla. Korjasin tämän ottamalla käyttöön hälytysten ryhmittelyn entiteetin mukaan.

![Oikea hyökkäys: incident suljettu](https://github.com/user-attachments/assets/6377fb31-1551-4aa1-9b93-4a00e9739fcd)

Muutama muu IP teki 1–2 yritystä (skannereita). Ne jäivät tarkoituksella molempien sääntöjen rajan alle.

## Säännöt koodina (CI/CD)

Tunnistussäännöt ovat repossa JSON-tiedostoina. Kun muutan sääntöä ja puskean sen `main`-haaraan, GitHub Actions tarkistaa tiedostot ja julkaisee ne Sentineliin.

```
[git push rules/*.json]
   │
   ▼
[GitHub Actions]  tarkistaa, että jokaisessa säännössä on tunniste, nimi ja kysely
   │  virheellinen → pysähtyy, mitään ei julkaista
   ▼
[Azure-kirjautuminen OIDC:llä]  lyhytikäinen tunniste, ei salasanaa
   │
   ▼
[deploy-rules.sh] → [Microsoft Sentinel]
```

- **Ei tallennettuja salasanoja:** GitHub saa jokaiselle ajolle lyhytikäisen tunnisteen, ja Azure hyväksyy sen vain tämän repon `main`-haarasta.
- **Vähimmät oikeudet:** GitHubin identiteetillä on vain *Microsoft Sentinel Contributor* -rooli labran resurssiryhmään. Se voi muokata sääntöjä, mutta ei koske virtuaalikoneeseen, verkkoon tai oikeuksiin.
- **Testattu päästä päähän:** muutin säännön 1 kuvauksen Gitissä, ja uusi teksti ja muokkausaika näkyivät Sentinelissä ilman yhtään käsin tehtyä vaihetta.
- **Miksi tämä on tärkeää:** jokainen säännön muutos jää Git-historiaan. Tunnistussäännön hiljainen heikentäminen on yksi tapa piilottaa hyökkäys, joten muutosten jäljitettävyys on tietoturvatiimissä vaatimus.

## Mitä opin

- **Ei-oletus ylläpitäjätunnus toimii:** hyökkääjä kokeili vain `Administrator`- ja `Admin`-tunnuksia, joten arvailu ei koskaan osunut oikeaan tiliin.
- **Tapahtumat eivät ole yrityksiä:** 10 yritystä tuotti 12 epäonnistumistapahtumaa. Kynnysarvot pitää mitoittaa sen mukaan.
- **Testit vaikuttavat toisiinsa:** 24 tunnin sääntö näkee vielä eilisen testin, joten testit pitää ajoittaa toistensa ikkunoiden ulkopuolelle.
- **Tilin lukitus on kompromissi:** se pysäyttää arvailun, mutta tunnuksen tietävä voi lukita tilin tahallaan (palvelunesto).
- **GitHubin OIDC-tunnisteen muoto:** ensimmäinen kirjautuminen epäonnistui, koska tunnisteessa oli repon nimen lisäksi sen pysyvä numerotunniste. Nimi voi vapautua toiselle, jos repo poistetaan, numero ei. Virheilmoitus näytti tarkan muodon, jolle luottamus piti antaa.
- **Vähimmissä oikeuksissa on aukkoja:** Sentinel-rooli saa muokata sääntöjä, mutta ei lukea itse Log Analytics -workspacea. Workflow antaa workspacen tunnisteen valmiina, joten laajempaa roolia ei tarvittu.
- **Väärä oletustilaus:** Azure CLI käytti aluksi toista tilausta, joten jokainen komento kiinnittää nyt `--subscription`-parametrin.

## Mitä tekisin nyt toisin

- Infrastruktuuri koodina Bicepillä Bash-skriptien sijaan
- Automaatiosääntö, joka sulkee omien testi-IP:ideni incidentit automaattisesti
- Säännöille testi, joka ajaa KQL-kyselyn ennen julkaisua eikä tarkista pelkkää tiedoston rakennetta

## Kustannukset

Budjettihälytys, VM:n automaattinen sammutus joka ilta, Sentinelin 31 päivän ilmaisjakso ja `teardown.sh`, joka poistaa kaiken. Koko labra rakentuu uudelleen noin 15 minuutissa komennolla `bash deploy.sh`, ja GitHubin identiteetti luodaan sen jälkeen komennolla `bash setup-ci.sh`.
