---
layout: post
title: "osTicket oppilaitokselle: VPN-rajattu julkaisu Proxmoxilla ja OPNsensellä"
date: 2025-12-08
excerpt: "Tukipyyntöjärjestelmä oppilaitoksen omaan infrastruktuuriin: Proxmox, Ubuntu-palvelin ja OPNsense. Internetistä palveluun pääsi vain WireGuard-VPN:n kautta, ja HTTPS-sertifikaatti uusiutui automaattisesti."
tags: [proxmox, ubuntu, opnsense, wireguard, caddy, lets-encrypt, apache, mariadb, linux]
---

**Lyhyesti:** Otin oppilaitoksessa käyttöön osTicket-tukipyyntöjärjestelmän omalla Linux-palvelimella ja suojasin sen niin, ettei palvelu näy julkiseen internetiin. Palvelin toimi Proxmox-virtualisointialustalla, ja internetistä palveluun pääsi vain WireGuard-VPN:n kautta. Yhteys salattiin HTTPS:llä, ja Let's Encrypt -sertifikaatin uusiminen automatisoitiin. Toteutin projektin itsenäisesti työharjoittelussa marras–joulukuussa 2025.

**Teknologiat:** Proxmox VE · Ubuntu Server 24.04 · Apache · MariaDB 10.11 · PHP 8.3 · osTicket 1.18.1 · OPNsense · WireGuard · Caddy · ACME/Let's Encrypt (DNS-01) · Dynamic DNS · UFW · OpenSSH

**Tausta:** Oppilaitoksen henkilökunta tarvitsi modernin tukipyyntöjärjestelmän. Aiempi järjestelmä toimi oppilaitoksen ulkopuolella yksityisellä palvelimella, mikä oli käytön ja ylläpidon kannalta epäkäytännöllistä. Uusi järjestelmä siirsi palvelun oppilaitoksen omaan infrastruktuuriin ja hallintaan.

---

## Tavoite

- Tukipyyntöjärjestelmä oppilaitoksen omaan infrastruktuuriin ja hallintaan
- Palvelu ei näy julkiseen internetiin: auki vain WireGuardin UDP-portti, pääsy internetistä vain VPN:n kautta
- Salattu yhteys ilman selaimen varoituksia ja ilman käsin tehtävää sertifikaattien uusimista

## Arkkitehtuuri

```
[Käyttäjä]
   │  WireGuard-tunneli (UDP, ainoa internetiin avattu portti)
   │  tunnelin sisällä HTTPS :8443
   ▼
[OPNsense]  palomuuri, WireGuard, Unbound-DNS,
            Caddy reverse proxy (TLS päättyy tähän, Let's Encrypt)
   │  HTTP :80, vain sisäverkossa
   ▼
[Ubuntu VM @ Proxmox]  Apache + PHP + MariaDB + osTicket
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
- Vaihdoin tietokannan salasanan vahvempaan ja päivitin sen osTicketin konfiguraatioon

### 4. Julkaisu VPN:n taakse (OPNsense)
- **Dynamic DNS** pitää domainin osoitteen ajan tasalla, kun operaattorin antama julkinen IP vaihtuu
- **Let's Encrypt DNS-01 -haasteella:** HTTP-01 vaatisi portin 80 auki internetiin, eikä sitä avattu. DNS-01 validoi domainin DNS-tietueen kautta, joten se toimii ilman avattuja web-portteja.
- **Caddy reverse proxy** porteissa 8080 (HTTP) ja 8443 (HTTPS), koska OPNsensen hallintapaneeli varaa portit 80 ja 443. Palomuuri päästää VPN:stä vain porttiin 8443, joten käyttäjien liikenne kulkee aina HTTPS:nä. TLS päättyy Caddyyn, joka välittää pyynnöt taustapalvelimelle sisäverkossa HTTP:nä (osTicket polussa `/osticket/`).
- **Unbound host override:** VPN-käyttäjillä domain osoittaa OPNsensen sisäiseen osoitteeseen, jolloin sama osoite ja sama sertifikaatti toimivat myös tunnelin sisällä
- **Palomuurisäännöt** WireGuard-rajapinnassa: palomuurille vain TCP 8443 (Caddy) ja DNS-portti 53 (jotta host override toimii)
- **WAN-rajapinnassa** auki vain WireGuardin UDP-portti; osTicketin portteja ei avattu internetiin

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

- **Tiedosto-oikeudet:** Asettaisin web-juuren omistajaksi rootin ja antaisin www-datalle kirjoitusoikeuden vain niihin hakemistoihin, joihin sovelluksen on pakko kirjoittaa. Näin sovellus ei pysty muokkaamaan omaa koodiaan. Palauttaisin `ost-config.php`:n vain luettavaksi heti asennuksen jälkeen.
- **Taustapalvelimen palomuuri:** Sallisin portin 80 vain OPNsensen osoitteesta, en koko sisäverkosta.
- **Caddyn käyttäjä:** Caddy oli ajossa oletuksena rootina (kuva yllä). Vaihtaisin sen vähäoikeuksiseen `www`-käyttäjään, jolloin mahdollinen murto Caddyn kautta ei anna heti koko palomuurin hallintaa.
- **Varmuuskopiot:** Ottaisin käyttöön Proxmoxin ajastetut VM-varmuuskopiot ja tietokannan dumpin sekä testaisin palautuksen.
- **Päivitykset:** Ottaisin käyttöön unattended-upgradesin tietoturvapäivityksille ja seuraisin osTicketin versiojulkaisuja.
- **Valvonta:** Asentaisin QEMU guest agentin ja perusvalvonnan levytilalle, muistille ja palvelun saatavuudelle.
- **SSH-avaimet:** Käyttäisin Ed25519-avaimia RSA-2048-avainten sijaan.
- **Levytila:** LVM-taltioryhmästä jäi oletusasetuksilla puolet varaamatta. Kasvattaisin juuritaltiota tai dokumentoisin varaamattoman tilan tietoisena valintana.

---

*Kuvakaappauksista on korvattu organisaation domain, julkinen IP-osoite ja muut tunnisteet esimerkkiarvoilla.*
