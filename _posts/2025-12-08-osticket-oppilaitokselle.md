---
layout: post
title: "osTicket oppilaitokselle: VPN-rajattu julkaisu Proxmoxilla ja OPNsensellä"
date: 2025-12-08
excerpt: "Tukipyyntöjärjestelmä oppilaitoksen omaan infrastruktuuriin: Proxmox-virtualisointi, Ubuntu-palvelin sekä OPNsense, jonka takaa palvelu julkaistiin vain WireGuard-VPN:n kautta Let's Encrypt -sertifikaatilla."
tags: [proxmox, ubuntu, opnsense, wireguard, caddy, lets-encrypt, apache, mariadb, linux]
---

**Lyhyesti:** Rakensin oppilaitokselle tukipyyntöjärjestelmän (osTicket) Proxmox-virtualisointialustalle. Palvelu on saavutettavissa vain WireGuard-VPN:n kautta, ja yhteys on salattu Let's Encrypt -sertifikaatilla, jonka uusiminen on automatisoitu. Toteutin projektin itsenäisesti marras–joulukuussa 2025.

**Teknologiat:** Proxmox VE · Ubuntu Server 24.04 · Apache · MariaDB 10.11 · PHP 8.3 · osTicket 1.18.1 · OPNsense · WireGuard · Caddy · ACME/Let's Encrypt (DNS-01) · Dynamic DNS · UFW · OpenSSH

**Tausta:** Oppilaitoksen henkilökunta tarvitsi modernin tukipyyntöjärjestelmän. Aiempi järjestelmä toimi oppilaitoksen ulkopuolella yksityisellä palvelimella, mikä oli käytön ja ylläpidon kannalta epäkäytännöllistä. Uusi järjestelmä siirsi palvelun oppilaitoksen omaan infrastruktuuriin ja hallintaan.

---

## Tavoite

- Tukipyyntöjärjestelmä oppilaitoksen omaan infrastruktuuriin ja hallintaan
- Ei porttiavausta julkiseen internetiin: pääsy vain VPN:n kautta
- Salattu yhteys ilman selaimen varoituksia ja ilman käsin tehtävää sertifikaattien uusimista

## Arkkitehtuuri

```
[Käyttäjä]
   │  WireGuard-VPN
   ▼
[OPNsense]  palomuuri, VPN-päätepiste, sisäinen DNS
   │  Caddy reverse proxy, HTTPS :8443 (Let's Encrypt)
   ▼
[Ubuntu VM @ Proxmox]  Apache + PHP + MariaDB
   │  HTTP :80, vain sisäverkossa
   ▼
[osTicket]
```

| Komponentti | Rooli |
|---|---|
| OPNsense | Palomuuri, WireGuard-päätepiste, sisäinen DNS (Unbound) |
| Caddy (OPNsense-plugin) | TLS-terminointi ja välitys taustapalvelimelle |
| ACME-client + DNS-01 | Let's Encrypt -sertifikaatti ja automaattinen uusiminen |
| Dynamic DNS | Pitää domainin ajan tasalla, kun julkinen IP vaihtuu |
| Ubuntu Server VM | osTicket (Apache, PHP, MariaDB) |

## Toteutus

### 1. Virtuaalikone ja käyttöjärjestelmä
- Ubuntu Server 24.04 -VM Proxmoxiin: 4 vCPU, 4 GiB RAM, 32 GiB levy, VirtIO SCSI -ohjain ja VirtIO-verkkokortti
- Kiinteä IP-osoite ja LVM-levyasettelu asennusvelhossa
- Järjestelmän päivitys heti asennuksen jälkeen

### 2. Sovelluspino
- Apache, MariaDB ja tarvittavat PHP-laajennukset (mysql, imap, intl, mbstring, gd, apcu, xml, zip, curl)
- osTicketille oma tietokanta ja oma tietokantakäyttäjä, jolle myönnettiin oikeudet vain tähän kantaan
- osTicket 1.18.1 asennettuna asennusvelhon kautta

### 3. Palvelimen koventaminen
- SSH: avainpohjainen kirjautuminen, salasanakirjautuminen pois, root-kirjautuminen estetty
- `~/.ssh` 700 ja `authorized_keys` 600
- UFW päälle, sallittuna vain SSH ja HTTP
- Tietokannan salasana vaihdettu vahvempaan ja päivitetty osTicketin konfiguraatioon

### 4. Julkaisu VPN:n taakse (OPNsense)
- **Dynamic DNS** pitää domainin osoitteen ajan tasalla, kun operaattorin antama julkinen IP vaihtuu
- **Let's Encrypt DNS-01 -haasteella:** HTTP-01 ei toimi, koska palvelimelle ei tule sisääntulevia yhteyksiä internetistä. DNS-01 validoi domainin DNS-tietueen kautta, joten se toimii ilman avattuja portteja.
- **Caddy reverse proxy** porteissa 8080/8443, koska OPNsensen hallintapaneeli varaa portit 80 ja 443. TLS päättyy Caddyyn, joka välittää pyynnön polusta `/osticket/*` taustapalvelimelle sisäverkossa HTTP:nä.
- **Unbound host override:** VPN-käyttäjillä domain osoittaa OPNsensen sisäiseen osoitteeseen, jolloin sama osoite ja sama sertifikaatti toimivat myös tunnelin sisällä
- **Palomuurisääntö** WireGuard-rajapinnassa: vain TCP 8443 palomuurille

![Dynamic DNS -asetus OPNsensessa](/kuvat/01-dyndns.png)

![Caddyn HTTP- ja HTTPS-portit](/kuvat/03-caddy-portit.png)

![Caddyn reverse proxy -sääntö](/kuvat/02-caddy-reverse-proxy.png)

## Vianmääritys: väärä sivusto portissa 80

Asennuksen jälkeen selain näytti osTicketin sijaan nginxin oletussivun, vaikka palvelimelle oli asennettu Apache. Etenin poissulkemalla:

1. **Pysäytin palvelimella ajossa olleen nginxin** ja varmistin `ss -lnp`-komennolla, että porttia 80 kuunteli enää Apache.
2. **Poistin web-juuresta nginxin jäljelle jättämät oletussivut.** Oletussivu näkyi silti.
3. **Vaihdoin VM:n IP-osoitteen,** ja osTicket alkoi näkyä. Vastaaja oli siis toinen laite, joka käytti samaa osoitetta.

**Opetus:** kiinteät osoitteet varataan ja kirjataan ylös ennen käyttöönottoa. Osoitekonfliktin olisi voinut todentaa heti esimerkiksi `arping -D` -komennolla tai vertaamalla ARP-taulun MAC-osoitetta VM:n omaan MAC-osoitteeseen, jolloin kaksi ensimmäistä vaihetta olisivat jääneet tarpeettomiksi.

## Testaus

| Testi | Tulos |
|---|---|
| Yhteys palveluun ilman VPN:ää (4G-verkosta) | Aikakatkaisu, palvelu ei vastaa julkisesta verkosta |
| Sertifikaatti selaimessa VPN:n kautta | Validi Let's Encrypt -sertifikaatti, ei varoituksia |
| Sertifikaatin uusiminen ACME-clientilla | Uusittu onnistuneesti, automaattinen uusiminen käytössä |
| Sovellus sisäverkosta suoraan | Toimii (HTTP vain sisäverkossa) |

![Let's Encrypt -sertifikaatti selaimessa](/kuvat/04-lets-encrypt.png)

![osTicketin käyttäjänäkymä](/kuvat/05-support-center.png)

## Luovutus

Luovutin valmiin järjestelmän ja teknisen dokumentaation oppilaitokselle joulukuussa 2025, ja ylläpito siirtyi oppilaitokselle. Luovutin samalla tunnukset, jotta oppilaitos pystyi vaihtamaan kaikki salasanat omikseen.

## Mitä tekisin nyt toisin

- **Tiedosto-oikeudet:** web-juuren omistajaksi root ja www-datalle kirjoitusoikeus vain niihin hakemistoihin, joihin sovelluksen on pakko kirjoittaa. Näin sovellus ei pysty muokkaamaan omaa koodiaan. `ost-config.php` palautetaan vain luettavaksi heti asennuksen jälkeen.
- **Taustapalvelimen palomuuri:** portti 80 sallittuna vain OPNsensen osoitteesta, ei koko sisäverkosta.
- **Varmuuskopiot:** Proxmoxin ajastetut VM-varmuuskopiot ja tietokannan dumppi, palautus testattuna.
- **Päivitykset:** unattended-upgrades tietoturvapäivityksille ja seuranta osTicketin versiojulkaisuille.
- **Valvonta:** QEMU guest agent sekä perusvalvonta levytilalle, muistille ja palvelun saatavuudelle.
- **SSH-avaimet:** Ed25519 RSA-2048:n sijaan.
- **Levytila:** LVM-taltioryhmästä jäi oletusasetuksilla puolet varaamatta. Kasvatettaisiin juuritaltiota tai jätettäisiin varaus tietoisena valintana dokumentoituna.

---

*Kuvakaappauksista on korvattu organisaation domain, julkinen IP-osoite ja muut tunnisteet esimerkkiarvoilla.*
