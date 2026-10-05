
## 1. Johdanto 
SNMP (Simple Network Management Protocol) on protokolla, jota käytetään verkkolaitteiden valvontaan ja hallintaan. SNMP:n avulla voidaan hakea laitteelta esimerkiksi prosessorin käyttöastetta, verkkoliikenteen määriä per aikayksikkö ja muita tietoja.

SNMP-agentti on laitteella, kuten palvelimella, reitittimellä tai kytkimellä, toimiva ohjelmisto, joka kerää tietoja laitteesta ja vastaa SNMP-managerin lähettämiin kyselyihin.

SNMP-manageri pyytää tietoa käyttämällä tiettyä OID-tunnistetta. SNMP-agentti palauttaa kyseiseen OID:hen liittyvän arvon.

**MIB (Management Information Base)** määrittelee, mitä yksiselitteisiä tietoja laitteesta voidaan SNMP:n avulla hakea. MIB kertoo, mitä tietoja laitteessa on saatavilla ja miten ne on järjestetty.

**OID (Object Identifier)** on yksittäisen MIB:ssä määritellyn tiedon yksilöllinen tunniste. Esimerkiksi CPU:n käyttöastetta kuvaavalla tiedolla voi olla oma OID.


## 2. Asennus

SNMP asennettiin seuraaville laitteille:

- web1
- db1
- branch-client
- ansible

Aluksi asennus tehtiin web1-palvelimelle ja myöhemmin muille. Kaikki asennukset aloitettiin kirjautumalla laitteiden bash-konsoliin ja päivittämällä apt-metakanta. Tosin ennen metakannan päivitystä piti lisätä DNS-nameserver osoitteet, jotta apt-update kehoitteet  toimivat. Se tehtiin seuraavalla komennolla:

 ```bash
printf 'nameserver 127.0.0.11\nnameserver 8.8.8.8\nnameserver 1.1.1.1\noptions ndots:0\n' > /etc/resolv.conf
```


Kaikille muille paitsi ansible-palvelimelle asennettiin SNMP-manageri ja -agentti seuraavalla komennolla:

```bash
apt install snmp snmpd -y
```

snmpd on daemon eli agentti, kun taas snmp sisältää managerin.

Ansiblelle asennettiin pelkästään SNMP-manageri:

```bash
apt install snmp -y
```

## SNMP-konfiguraatio

Web1-palvelimen konfiguraatiotiedostoon lisättiin luennolla läpikäydyn asennusesimerkin mukaisesti:

```text
view systemonly included .1.3.6.1.2
```

Konfiguraatiotiedostosta snmpd.conf kommentoitiin myös `mibs`-asetus, jolloin MIB-tunnisteet saadaan muunnettua ihmisten helpommin luettaviksi nimiksi.


Näiden toimien jälkeen tehtiin ensimmäinen yhteyskokeilu:

```bash
snmpwalk -v2c public web1 system
```

Tämä ei vielä toiminut.

Selvitin ongelmaa käyttäen ChatGPT:tä ja sieltä saamieni vinkkien perusteella muokkasin web1-, db1- ja branch-client-palvelimien `snmpd.conf`-tiedostosta `agentaddress`-asetusta.

Vanha asetus oli:

```text
agentaddress 127.0.0.1,[::1]
```

ja se muutettiin muotoon:

```text
:upd:161
```

Tämän lisäksi ansible-palvelimella ajettiin:

```bash
apt install snmp-mibs-downloader -y
download-mibs
```

Tämän seurauksena MIB asentui hakemistoon:

```text
/var/lib/mibs/ietf/SNMPv2-MIB
```

Ongelma ratkaistiin lopulta lisäämällä SNMP:n `snmp.conf`-konfiguraatiotiedoston loppuun:

```text
mibdirs /var/lib/mibs/ietf:/var/lib/mibs/iana:/usr/share/snmp/mibs
```

Tämän jälkeen seuraavat komennot alkoivat toimia:

```bash
snmpwalk -v2c -c public web1 system
snmpwalk -v2c -c public db1 system
snmpwalk -v2c -c public branch-client system
```

---

## 3. Kerätyt tiedot 

## Järjestelmän nimi

Järjestelmän nimi on `web1`.

Tieto haettiin komennolla:

```bash
snmpget -v2c -c public web1 sysName.0
```

Tuloste:

```text
root@ansible:/# snmpget -v2c -c public web1 sysName.0
SNMPv2-MIB::sysName.0 = STRING: web1
```

## Järjestelmän kuvaus

Järjestelmän kuvaus:

- Linux
- Ubuntu 22.04.1
- Kernel 6.8.0-138-generic
- x86_64-arkkitehtuuri

Tieto haettiin komennolla:

```bash
snmpget -v2c -c public web1 sysDescr.0
```

Tuloste:

```text
root@ansible:/# snmpget -v2c -c public web1 sysDescr.0
SNMPv2-MIB::sysDescr.0 = STRING: Linux web1 6.8.0-138-generic #138~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 7 13:43:15 UTC x86_64
```

## Käyttöaika

SNMP:n ilmoittama käyttöaika oli:

```text
5:28:24.75
```

Tieto haettiin komennolla:

```bash
snmpget -v2c -c public web1 sysUpTime.0
```

Tuloste:

```text
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (1970475) 5:28:24.75
```

## 4. Verkkorajapinnat

Web1-palvelimelta löytyy kaksi rajapintaa:

- `lo` – local-rajapinta, jota käytetään laitteen sisäiseen liikenteeseen.
- `eth0` – Ethernet-verkkorajapinta, joka yhdistää laitteen verkkoon.

Rajapinnat löytyivät komennolla:

```bash
snmpwalk -v2c -c public web1 ifDescr
```

Tuloste:

```text
root@ansible:/# snmpwalk -v2c -c public web1 ifDescr
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.2 = STRING: eth0
```

## 5. OID-analyysi

| OID | Tarkoitus |
|---|---|
| `sysName.0` | Laitteen nimi, esimerkiksi `web1`. |
| `sysDescr.0` | Tekstimuotoinen kuvaus laitteen raudasta ja käyttöjärjestelmästä. |
| `sysUpTime.0` | Sadasosasekunnin tarkkuudella annettu aika siitä, kun SNMP:n hallintaosa viimeksi käynnistettiin. |
| `ifDescr` | Tekstimuotoinen kuvaus verkkorajapinnasta. Voi sisältää esimerkiksi valmistajan, tuotteen nimen ja laitteisto-/ohjelmistoversion. |
| `ifOperStatus` | Verkkorajapinnan tämänhetkinen tila numerona, esimerkiksi Up (1), Down (2), Testing (3) tai Dormant (5). |

## OID:ien kuvaukset

### sysName.0

`sysName.0` sisältää hallinnollisesti määritellyn nimen laitteelle. Käytännössä tämä tarkoittaa esimerkiksi laitteen nimeä:

```text
web1
```

### sysDescr.0

`sysDescr.0` sisältää tekstimuotoisen kuvauksen laitteesta. Siinä voi olla tietoja esimerkiksi:

- laitteistosta
- käyttöjärjestelmästä
- käyttöjärjestelmän versiosta
- verkkokomponenteista

### sysUpTime.0

`sysUpTime.0` kertoo ajan sadasosasekunteina siitä, kun järjestelmän SNMP:n hallintaosa viimeksi käynnistettiin uudelleen.

### ifDescr

`ifDescr` sisältää tekstimuotoisen kuvauksen verkkorajapinnasta.

Esimerkiksi:

```text
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.2 = STRING: eth0
```

### ifOperStatus

`ifOperStatus` kertoo verkkorajapinnan tämänhetkisen toiminnallisen tilan.

Esimerkiksi:

- `up(1)` – rajapinta on käytössä
- `down(2)` – rajapinta ei ole käytössä
- `testing(3)` – rajapinta on testitilassa
- `dormant(5)` – rajapinta odottaa jotain ulkoista tapahtumaa

---

# Useamman laitteen valvonta

SNMP:n avulla haettiin tiedot alla olevaan taulukkoon:

| Laite | Nimi | Käyttöjärjestelmä | Uptime |
|---|---|---|---|
| Web1 | web1 | Linux, Ubuntu 22.04.1, kernel 6.8.0-138-generic, x86_64 | 8:08:44.53 |
| Db1 | db1 | Linux, Ubuntu 22.04.1, kernel 6.8.0-138-generic, x86_64 | 0:23:42.38 |
| branch-client | branch-client | Linux, Ubuntu 22.04.1, kernel 6.8.0-138-generic, x86_64 | 0:00:56.08 |

---

#### Taulukon tietojen hankintaan käytetyt tulosteet

 ##### branch-client

Komento:

```bash
snmpwalk -v2c -c public branch-client system
```

Tuloste:

```text
root@ansible:/# snmpwalk -v2c -c public branch-client system
SNMPv2-MIB::sysDescr.0 = STRING: Linux branch-client 6.8.0-138-generic #138~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 7 13:43:15 UTC x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (5608) 0:00:56.08
SNMPv2-MIB::sysContact.0 = STRING: Me <me@example.org>
SNMPv2-MIB::sysName.0 = STRING: branch-client
SNMPv2-MIB::sysLocation.0 = STRING: Sitting on the Dock of the Bay
SNMPv2-MIB::sysServices.0 = INTEGER: 72
SNMPv2-MIB::sysORLastChange.0 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORID.1 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
SNMPv2-MIB::sysORID.2 = OID: SNMP-MPD-MIB::snmpMPDCompliance
SNMPv2-MIB::sysORID.3 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.5 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.6 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.7 = OID: UDP-MIB::udpMIB
SNMPv2-MIB::sysORID.8 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
SNMPv2-MIB::sysORID.10 = OID: NOTIFICATION-LOG-MIB::notificationLogMIB
SNMPv2-MIB::sysORDescr.1 = STRING: The SNMP Management Architecture MIB.
SNMPv2-MIB::sysORDescr.2 = STRING: The MIB for Message Processing and Dispatching.
SNMPv2-MIB::sysORDescr.3 = STRING: The management information definitions for the SNMP User-based Security Model.
SNMPv2-MIB::sysORDescr.4 = STRING: The MIB module for SNMPv2 entities
SNMPv2-MIB::sysORDescr.5 = STRING: View-based Access Control Model for SNMP.
SNMPv2-MIB::sysORDescr.6 = STRING: The MIB module for managing TCP implementations
SNMPv2-MIB::sysORDescr.7 = STRING: The MIB module for managing UDP implementations
SNMPv2-MIB::sysORDescr.8 = STRING: The MIB module for managing IP and ICMP implementations
SNMPv2-MIB::sysORDescr.9 = STRING: The MIB modules for managing SNMP Notification, plus filtering.
SNMPv2-MIB::sysORDescr.10 = STRING: The MIB module for logging SNMP Notifications.
SNMPv2-MIB::sysORUpTime.1 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.2 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.3 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.4 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.5 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.6 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.7 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.8 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.9 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.10 = Timeticks: (0) 0:00:00.00
```

##### db1

Komento:

```bash
snmpwalk -v2c -c public db1 system
```

Tuloste:

```text
root@ansible:/# snmpwalk -v2c -c public db1 system
SNMPv2-MIB::sysDescr.0 = STRING: Linux db1 6.8.0-138-generic #138~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 7 13:43:15 UTC x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (142238) 0:23:42.38
SNMPv2-MIB::sysContact.0 = STRING: Me <me@example.org>
SNMPv2-MIB::sysName.0 = STRING: db1
SNMPv2-MIB::sysLocation.0 = STRING: Sitting on the Dock of the Bay
SNMPv2-MIB::sysServices.0 = INTEGER: 72
SNMPv2-MIB::sysORLastChange.0 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORID.1 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
SNMPv2-MIB::sysORID.2 = OID: SNMP-MPD-MIB::snmpMPDCompliance
SNMPv2-MIB::sysORID.3 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.5 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.6 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.7 = OID: UDP-MIB::udpMIB
SNMPv2-MIB::sysORID.8 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
SNMPv2-MIB::sysORID.10 = OID: NOTIFICATION-LOG-MIB::notificationLogMIB
SNMPv2-MIB::sysORDescr.1 = STRING: The SNMP Management Architecture MIB.
SNMPv2-MIB::sysORDescr.2 = STRING: The MIB for Message Processing and Dispatching.
SNMPv2-MIB::sysORDescr.3 = STRING: The management information definitions for the SNMP User-based Security Model.
SNMPv2-MIB::sysORDescr.4 = STRING: The MIB module for SNMPv2 entities
SNMPv2-MIB::sysORDescr.5 = STRING: View-based Access Control Model for SNMP.
SNMPv2-MIB::sysORDescr.6 = STRING: The MIB module for managing TCP implementations
SNMPv2-MIB::sysORDescr.7 = STRING: The MIB module for managing UDP implementations
SNMPv2-MIB::sysORDescr.8 = STRING: The MIB module for managing IP and ICMP implementations
SNMPv2-MIB::sysORDescr.9 = STRING: The MIB modules for managing SNMP Notification, plus filtering.
SNMPv2-MIB::sysORDescr.10 = STRING: The MIB module for logging SNMP Notifications.
SNMPv2-MIB::sysORUpTime.1 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.2 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.3 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.4 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.5 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.6 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.7 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.8 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.9 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.10 = Timeticks: (0) 0:00:00.00
```

##### web1

Komento:

```bash
snmpwalk -v2c -c public web1 system
```

Tuloste:

```text
root@ansible:/# snmpwalk -v2c -c public web1 system
SNMPv2-MIB::sysDescr.0 = STRING: Linux web1 6.8.0-138-generic #138~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 7 13:43:15 UTC x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (2932453) 8:08:44.53
SNMPv2-MIB::sysContact.0 = STRING: Me <me@example.org>
SNMPv2-MIB::sysName.0 = STRING: web1
SNMPv2-MIB::sysLocation.0 = STRING: Sitting on the Dock of the Bay
SNMPv2-MIB::sysServices.0 = INTEGER: 72
SNMPv2-MIB::sysORLastChange.0 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORID.1 = OID: SNMP-FRAMEWORK-MIB::snmpFrameworkMIBCompliance
SNMPv2-MIB::sysORID.2 = OID: SNMP-MPD-MIB::snmpMPDCompliance
SNMPv2-MIB::sysORID.3 = OID: SNMP-USER-BASED-SM-MIB::usmMIBCompliance
SNMPv2-MIB::sysORID.4 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.5 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.6 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.7 = OID: UDP-MIB::udpMIB
SNMPv2-MIB::sysORID.8 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.9 = OID: SNMP-NOTIFICATION-MIB::snmpNotifyFullCompliance
SNMPv2-MIB::sysORID.10 = OID: NOTIFICATION-LOG-MIB::notificationLogMIB
SNMPv2-MIB::sysORDescr.1 = STRING: The SNMP Management Architecture MIB.
SNMPv2-MIB::sysORDescr.2 = STRING: The MIB for Message Processing and Dispatching.
SNMPv2-MIB::sysORDescr.3 = STRING: The management information definitions for the SNMP User-based Security Model.
SNMPv2-MIB::sysORDescr.4 = STRING: The MIB module for SNMPv2 entities
SNMPv2-MIB::sysORDescr.5 = STRING: View-based Access Control Model for SNMP.
SNMPv2-MIB::sysORDescr.6 = STRING: The MIB module for managing TCP implementations
SNMPv2-MIB::sysORDescr.7 = STRING: The MIB module for managing UDP implementations
SNMPv2-MIB::sysORDescr.8 = STRING: The MIB module for managing IP and ICMP implementations
SNMPv2-MIB::sysORDescr.9 = STRING: The MIB modules for managing SNMP Notification, plus filtering.
SNMPv2-MIB::sysORDescr.10 = STRING: The MIB module for logging SNMP Notifications.
SNMPv2-MIB::sysORUpTime.1 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.2 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.3 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.4 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.5 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.6 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.7 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.8 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.9 = Timeticks: (0) 0:00:00.00
SNMPv2-MIB::sysORUpTime.10 = Timeticks: (0) 0:00:00.00
```


# Omat havainnot SNMP:n hyödyistä ja rajoituksista

SNMP vaikuttaa suoraviivaiselta ja  toimivalta keinolta kerätä dataa eri verkkolaitteilta kirjautumatta suoraan ko. laitteille. Käyttöönoton kanssa oli hieman haasteita, mutta ne taitavat kuulua asiaan.  Tiedonkeruun automatisointi ja automaattiset varoitukset helpottaisivat verkossa ilmenevien ongelmien havaitsemista ja tarvittavaa reagoimista. 

Luennolla kuultua tietoturvaan liittyen: 

SNMP:n avulla voidaan kerätä keskitetysti paljon tietoa eri verkkolaitteista. Samalta hallintapalvelimelta voidaan esimerkiksi tarkistaa useiden palvelimien nimet, käyttöajat, käyttöjärjestelmät sekä verkkorajapintojen tilat.

SNMPv2c:n selkeä rajoitus on tietoturva. SNMPv2c ei sisällä salattua dataa, vaan siirtää tiedot verkossa salaamattomana. Tämän takia liikennettä pystyy lukemaan henkilö, joka on pääsee lukemaan verkkoliikennettä.

SNMPv3 parantaa tietoturvaa, koska  mahdollistaa datan kryptaamisen sekä käyttäjätunnuksen ja salasanan käytön. Lisäksi SNMPv3:ssa voidaan käyttää autorisaatiota eli määritellä eri käyttäjille erilaisia käyttöoikeuksia tietoihin.




Tekoälyn käyttö:
Käytän harjoitusten tekemiseen Linux-konetta, jossa on Ubuntu 22.04.5, en tiedä johtuiko tästä vai jostain muusta, että MIB toiminnan kanssa oli ongelmaa. Näiden ongelmien selvittämiseen käytin Chat-GPT:tä (versio GPT-5.6 Sol) copy+pastettamalla virheilmoituksia Bashista Chat-GPT promptiin.  
Tiedustelin Chat GPT:ltä myös miten Markdownissa saa myös Githubin puolella näkyvät laatikot bash-komentojen ympärille.







