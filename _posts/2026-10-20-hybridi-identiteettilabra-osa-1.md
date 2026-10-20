---
layout: post
title: "Hybridi-identiteettilabra, osa 1: Active Directory ja käyttäjien elinkaari PowerShellillä"
date: 2026-10-20
excerpt: "Hyper-V-labra, jossa Windows Server 2025 -toimialueohjain ja PowerShell-automaatio käyttäjien käyttöönottoon ja käytöstä poistoon. HelpDeskin oikeudet rajattu vähimpiin ja testattu. Osassa 2 ympäristö synkronoidaan Entra ID:hen."
tags: [active-directory, powershell, windows-server, hyper-v, identity-management, least-privilege, entra-id]
---

**Lyhyesti:** Rakensin kotikoneelle Hyper-V-labran, jossa on Windows Server 2025 -toimialueohjain (DC, palvelin joka hallitsee toimialueen tunnuksia ja kirjautumisia). Automatisoin käyttäjien elinkaaren PowerShellillä: uudet työntekijät luodaan CSV-tiedostosta, ja lähtijät poistetaan käytöstä työntekijänumeron perusteella. HelpDesk-ryhmä voi nollata salasanoja ja avata lukituksia vain aktiivisilta käyttäjiltä. Tämä on sarjan ensimmäinen osa, ja toisessa osassa ympäristö synkronoidaan Microsoft Entra ID:hen. Toteutin projektin itsenäisesti lokakuussa 2026.

**Teknologiat:** Windows Server 2025 · Active Directory Domain Services · DNS · Hyper-V · PowerShell · PowerShell Direct · Git · gitleaks

**Koodi:** [github.com/Tuomas404/hybrid-identity-lab](https://github.com/Tuomas404/hybrid-identity-lab)

---

## Tavoite

- Harjoitella tehtäviä, joita IT-tuessa ja ylläpidossa tehdään päivittäin: käyttäjien luonti, käytöstä poisto ja salasanojen nollaus
- Skriptit, jotka voi ajaa turvallisesti uudelleen ja joiden muutokset voi esikatsella ennen tekemistä
- Oikeudet rajattuna vähimpiin: HelpDesk saa vain sen, mitä työ vaatii
- Pohja hybridi-identiteetille, jossa AD:n käyttäjät näkyvät myöhemmin myös Entra ID:ssä

## Arkkitehtuuri

```
[Windows 11 Pro -isäntäkone]  Hyper-V, Git-repo ja skriptit
   │  PowerShell Direct: skriptit ajetaan VM:ssä ilman verkkoyhteyttä
   ▼
[DC01: Windows Server 2025]  AD DS + DNS, toimialue lab.test, 10.10.10.10
   │
   ├──▶ [CL01: Windows 11 -testikone]  osa 2
   │
   ▼
[Microsoft Entra ID]  synkronointi, osa 2
```

| Komponentti | Rooli |
|---|---|
| Hyper-V + NAT-kytkin | Erillinen labraverkko `10.10.10.0/24`, internetyhteys isäntäkoneen kautta |
| DC01 | Toimialueohjain: tunnukset, ryhmät, kirjautuminen ja DNS |
| PowerShell-skriptit | OU-rakenne, ryhmät, käyttöönotto, käytöstä poisto, raportti ja delegointi |
| Git + GitHub | Skriptien versiohallinta; salasanat ja raportit pidetään repon ulkopuolella |

## OU-rakenne

OU (organisaatioyksikkö) on AD:n kansio, jonka avulla tunnuksiin kohdistetaan asetuksia ja oikeuksia.

```
lab.test
└── LAB
    ├── Users
    │   ├── HR
    │   ├── IT
    │   └── Sales
    ├── Computers
    │   ├── Workstations
    │   └── Servers
    ├── Groups
    ├── ServiceAccounts
    └── Disabled
```

- **Ylätaso objektityypin mukaan:** ryhmäkäytännöt (GPO, keskitetysti jaettavat asetukset), oikeuksien delegointi ja Entra-synkronoinnin rajaus voidaan kohdistaa erikseen käyttäjiin ja koneisiin.
- **Osastot vain Usersin alla:** osastotieto on myös ryhmissä, joten koneita ei tarvitse jakaa osastoittain.
- **Disabled on Usersin rinnalla, ei sen alla:** tällä on merkitystä oikeuksien kannalta (ks. delegointi).

## Käyttäjien elinkaari

| Skripti | Tehtävä |
|---|---|
| `New-LabOU.ps1`, `New-LabGroups.ps1` | OU-rakenne sekä osasto- ja roolikohtaiset ryhmät |
| `New-LabUsers.ps1` | Käyttöönotto CSV:stä: tunnus, alkusalasana, osaston OU ja ryhmä |
| `Disable-LabUser.ps1` | Käytöstä poisto työntekijänumerolla |
| `Get-LabUserReport.ps1` | Raportti tunnuksista auditointia varten (tila, OU, ryhmät, kirjautumis- ja salasanatiedot) |
| `Grant-LabHelpDeskRights.ps1` | HelpDeskin oikeudet |

Tärkeimmät ratkaisut:

- **Työntekijänumero tunnisteena:** nimi ei yksilöi (testidatassa on tarkoituksella kaksi Mikko Virtasta) eikä pysy samana, ja käyttäjätunnus syntyy vasta skriptissä. Sama ajatus on Entra-synkronoinnin source anchorissa (pysyvä tunniste, jolla AD- ja Entra-tunnus yhdistetään).
- **Käyttäjätunnukset:** ääkköset muutetaan (Mäkinen → `makinen`), pituus enintään 20 merkkiä, ja varattuun tunnukseen lisätään numero (`mikko.virtanen2`).
- **Alkusalasana:** satunnainen 16 merkkiä ja vaihto ensimmäisellä kirjautumisella. Salasanat tallennetaan repon ulkopuolelle.
- **Käytöstä poisto ei poista tunnusta:** tunnus estetään, kuvaukseen tulee päivämäärä, ryhmät poistetaan ja tunnus siirretään `OU=Disabled`. Tunnus säilyy, jos tiedostoja tai oikeuksia pitää myöhemmin selvittää.
- **Idempotenssi** (skriptin voi ajaa monta kertaa, ja lopputulos pysyy samana): skripti tarkistaa ensin tilan ja tekee vain puuttuvat vaiheet.
- **`-WhatIf`:** näyttää, mitä tehtäisiin, muuttamatta mitään.

```powershell
# Collect only the steps that are still missing (idempotency)
$actions = @()
if ($u.Enabled)              { $actions += 'Disable' }
if ($groupNames.Count -gt 0) { $actions += 'RemoveGroups' }
if (-not $inDisabledOU)      { $actions += 'MoveToDisabled' }

if ($actions.Count -eq 0) { $result.Status = 'AlreadyDone'; $result; continue }
```

Skripti palauttaa jokaisesta tunnuksesta tilan tekstin sijaan, joten tulokset voi suodattaa tai tallentaa. Kaikki tilat on testattu:

| Tilanne | Tulos |
|---|---|
| Aktiivinen tunnus | `Offboarded` |
| Sama tunnus ajetaan uudelleen | `AlreadyDone` |
| Tuntematon työntekijänumero | `NotFound` |
| Sama numero kahdella tunnuksella | `Failed` (ei tehdä mitään) |
| Ajo parametrilla `-WhatIf` | `WhatIf` |

## HelpDeskin oikeudet (delegointi)

Delegointi tarkoittaa, että annetaan rajattu ylläpito-oikeus ilman ylläpitäjän tunnusta.

- **Mitä HelpDesk saa:** ryhmä `SG-Role-HelpDesk` voi `OU=Users`-kansion käyttäjille nollata salasanan, avata lukituksen ja asettaa vaatimuksen vaihtaa salasana seuraavalla kirjautumisella. Muuta se ei voi tehdä.
- **Oikeudet ryhmälle, ei henkilöille:** kun rooli vaihtuu, ryhmästä poistaminen poistaa kaikki oikeudet kerralla.
- **Testattu HelpDesk-testitunnuksella:**
  - Lukituksen avaus ja salasanan nollaus `OU=Usersissa`: onnistui
  - Salasanan nollaus `OU=Disabledissa`: estetty
  - Käyttäjän kuvauksen muuttaminen: estetty
- **Miksi Disabled on Usersin rinnalla:** oikeus periytyy vain Usersin alle, joten lähteneen työntekijän tunnusta ei voi ottaa uudelleen käyttöön salasanan nollauksella.
- **Skripti eikä ohjattu toiminto:** Windowsin Delegation of Control -ohjattu toiminto ei jätä jälkeä siitä, mitä annettiin. Skripti on toistettava ja versiohallittu. Liian laajat delegoinnit ovat yleinen AD-hyökkäysreitti.

## Tarkistukset

- **Toimialueohjaimen kunto:** DNS-nimet ratkeavat, SYSVOL- ja NETLOGON-jaot ovat olemassa, ja DFSR-tapahtuma 4602 (SYSVOL-kansion replikointi käynnistyi) löytyy.
- **Ennen repon julkaisua:** koko Git-historian tiedostolista ja gitleaks-skannaus (työkalu, joka etsii salasanoja ja avaimia commiteista). Kumpikin tarvitaan, koska gitleaks ei tunnista esimerkiksi CSV-rivillä olevaa satunnaista salasanaa.

## Mitä opin

- **Toimialueohjaimen DNS osoittaa itseensä:** jos se osoittaisi julkiseen DNS:ään, DC ei löytäisi omia SRV-tietueitaan, ja kirjautuminen, toimialueeseen liittyminen ja ryhmäkäytännöt lakkaisivat toimimasta.
- **Verkkoprofiili jäi Public-tilaan:** verkon tunnistava palvelu (NLA) käynnistyi ennen AD:ta eikä yrittänyt uudelleen. Viivästetty käynnistys korjasi sen. Väärä profiili tarkoittaa vääriä palomuurisääntöjä.
- **Ääkköset:** Windows PowerShell 5.1 lukee UTF-8-tiedoston ilman BOM-merkkiä väärällä merkistöllä (`MÃ¤kinen`). Skriptit ovat siksi pelkkää ASCIIa, ja CSV luetaan `-Encoding UTF8`.
- **Nimen pitää olla uniikki saman OU:n sisällä:** kaksi Mikko Virtasta eri osastoilta törmäsi, kun molemmat siirrettiin Disabled-kansioon. Skripti lisää nyt nimeen työntekijänumeron ennen siirtoa.
- **Salasanan nollaus poistaa vaihtovaatimuksen:** HelpDeskin pitää asettaa "vaihda salasana seuraavalla kirjautumisella" uudelleen, muuten se tietää käyttäjän pysyvän salasanan.
- **Käyttämätön tunnus alkusalasanalla on riski:** alkusalasanan tietää muukin kuin käyttäjä, ja ensimmäinen kirjautuja saa tunnuksen. Parempi on poistaa tunnus automaattisesti käytöstä, jos sitä ei oteta käyttöön, tai käyttää Entran kertakäyttöistä pääsykoodia (Temporary Access Pass).

## Mitä tekisin nyt toisin

- Erilliset ylläpitäjätunnukset (`adm-`) omaan OU:hun alusta asti, jolloin ylläpito on erotettu arkikäytöstä
- AD Recycle Bin (poistettujen objektien palautus) päälle heti asennuksen jälkeen
- Automaattiset testit (Pester) ja koodin tarkistus (PSScriptAnalyzer) GitHub Actionsiin
- Isossa organisaatiossa käyttäjät luotaisiin suoraan HR-järjestelmästä Entra ID:hen. CSV ja PowerShell sopivat pieniin organisaatioihin ja ovat yhä yleisiä.

## Seuraavaksi (osa 2)

- Windows 11 -testikone toimialueeseen ja ryhmäkäytännöt
- Synkronointi Entra ID:hen, MFA ja ehdollinen pääsy (Conditional Access)
- Entra ID:n kirjautumislokit Microsoft Sentineliin, mikä yhdistää labran [honeypot-projektiini]({% post_url 2026-10-18-azure-sentinel-honeypot %})

## Kustannukset

0 €. Hyper-V sisältyy Windows 11 Prohon, ja Windows Server -lisenssi on Azure Education Hubin opiskelijalisenssi.
