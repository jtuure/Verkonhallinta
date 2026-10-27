## 1. Johdanto

Selitä Infrastructure as code (IaC). 

Perusajatus on, että hallitaan tavoitetilaa, ei yksittäisiä komentokertoja. Toisin sanoen hallitaan laajempaa infrastruktuuria koodin avulla sen sijaan, että palvelimien ja muiden laitteiden asetuksia ja määrityksiä tehtäisiin käsin.  Infrastruktuurin määritykset ovat luettavaa, toistettavaa ja versiohallittavaa aineistoa. IaC:n etuna on prosessien automatisointi, toistettavuus ja versionhallinta. IaC periaatteella toimiminen on myös nopeampaa ja vähemmän virhealtista, kuin esim. palvelinmääritysten tekeminen käsin. 
## 2. Inventory


Inventory saatiin näkyville ajamalla ansible-palvelimella bash-terminaalissa komento:

```bash
cat /ansible/inventory.ini
```
Tuloksena oli inventory, josta löytyi Taulukon 1 mukaiset laitteet ryhmiteltynä.

*Taulukko 1. Inventoryn ryhmät sekä ryhmiin kuuluvat laitteet*
 
| Ryhmä     | Laitteet
|-----------|---------------------------------------|
| routers   | r1, r2, r3                            |
| clients   | client1, attacker, branch-client      |
| servers   | web1, db1                             |
| monitoring| prometheus, grafana, zabbix, cadvisor |
| management| ansible                               |

Ryhmät ovat muodostettu laitteiden käyttötarkoituksen mukaan. Esim. routers, clients servers. Toisaalta löytyy myös ryhmittelyä sen mukaan missä verkossa laitteet ovat (Network segment) ja mihin infrastruktuurin ryhmään (infrastructure group) laitteet kuuluvat. Eri ryhmittelyt auttavat tunnistamaan minkä tyyppisiä laitteita verkkoympäristössä on sekä myös tunnistamaan missä verkkotopologian osassa laitteet ovat. 


Seuraavaksi tarkistettiin yhteydet ajamalla komento:
```bash
ansible all -i /ansible/inventory.ini -m ping
```
Ansible-palvelimelta sai kaikkiin kolmeen reitittimeen (r1, r2, r3).

r3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
r1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
r2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

Muiden laitteiden kohdalla luki "UNREACHABLE". 

## 3. SNMP Playbook

## 4. Node Exporter Playbook

## 5. Vertailu

## 6. Yhteenveto
