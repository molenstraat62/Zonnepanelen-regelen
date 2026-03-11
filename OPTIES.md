# Opties voor het regelen van je JFY Sunleaf 2500TL op basis van Youless LS120 data

Omdat de salderingsregeling in 2027 afgeschaft wordt, is het een heel goed idee om je eigen zonnestroom-overschot te minimaliseren door je omvormer actief te "knijpen" (het uitgaande vermogen reduceren) in plaats van hem simpelweg aan of uit te schakelen. Aangezien je Home Assistant draait, heb je hiervoor de perfecte basis.

Hieronder staan een aantal concrete, lokaal werkende en veilige opties uitgewerkt. Omdat JFY (JingFuYuan) omvormers vaak gebruik maken van gestandaardiseerde Chinese omvormer-hardware (vaak met Solarman-gebaseerde wifi-modules), zijn er verschillende manieren om dit aan te pakken zonder dat je afhankelijk bent van de cloud.

---

## De Basis: Youless LS120 in Home Assistant
Voordat je de omvormer kunt aansturen, moet Home Assistant weten hoeveel je teruglevert.
De Youless LS120 is gelukkig lokaal uitstekend te integreren, en dit kan zelfs parallel aan Enelogic blijven werken.

1. Installeer in Home Assistant de officiële **Youless integratie** via `Instellingen > Apparaten & Diensten`.
2. Vul het IP-adres van je Youless in.
3. Je krijgt nu entiteiten zoals `sensor.power_delivered` (stroom die je aan het net levert) en `sensor.power_consumed` (stroom die je van het net trekt).
4. Deze waarden ga je gebruiken in een Automatisering of in Node-RED om te bepalen hoeveel de JFY Sunleaf "geknepen" moet worden. Als `power_delivered` bijvoorbeeld 1000W is en je wilt 0 terugleveren, moet de omvormer 1000W minder gaan produceren.

---

## Optie 1: Lokaal via de bestaande WiFi-module (Modbus TCP)
Veel JFY-omvormers gebruiken WiFi-dongles die onder de motorkap gebaseerd zijn op de **Solarman** chipsets (zoals Deye, Growatt, en Sofar Solar). Als dit het geval is, staat poort `8899` (soms `502`) lokaal open op het IP-adres van de omvormer. Hierover praat de omvormer het Modbus TCP of Solarman V5 protocol.

**Hoe dit werkt:**
1. Zoek in Home Assistant (via HACS) naar de integratie **Solarman** (of eventueel **Modbus** als het pure Modbus TCP is).
2. Je kunt proberen de omvormer toe te voegen met het IP-adres, het serienummer van de logger (staat vaak op de WiFi-dongle) en poort 8899.
3. Als dit lukt, kun je de omvormer lokaal uitlezen.
4. **Het knijpen:** Als de integratie succesvol is, zoek je naar het Modbus-register voor de **Active Power Limit** (vaak register `40` of in de `100+` range, afhankelijk van de exacte firmware). Via een service call in Home Assistant (`solarman.write_holding_register` of `modbus.write_register`) kun je dit percentage of absolute vermogen dynamisch aanpassen.

**Voordeel:** Je hoeft geen extra hardware te kopen en hoeft geen kabels te trekken.
**Nadeel:** Omdat je je zorgen maakt over security: de module praat waarschijnlijk ook met een Chinese cloud.
**Security tip:** Je kunt in je router de WiFi-module van de omvormer blokkeren voor internettoegang ("Parental Control" of Firewall rule). Zolang Home Assistant in hetzelfde lokale netwerk (of VLAN) zit, blijft de lokale communicatie via poort 8899 gewoon werken, maar kan de omvormer niet meer naar buiten bellen.

---

## Optie 2: Hard-wired lokale communicatie (RS485/RS232 naar WiFi of Ethernet)
Wil je de originele WiFi-module er helemaal uithalen (omdat je de cloud-connectie of de security niet vertrouwt), dan kun je de seriële poort van de JFY Sunleaf (vaak RS232 of RS485) direct uitlezen.

**Hoe dit werkt:**
1. Je verwijdert de JFY WiFi-module.
2. In de poort (of via een interne klemmenstrook) sluit je een **Elfin EW11** (RS485 naar WiFi), **Elfin EE11** (RS485 naar Ethernet) of een zelfgebouwde **ESP32 met een MAX485 module** aan.
3. Deze modules vertalen de seriële (Modbus RTU) signalen van de omvormer naar Modbus TCP over je eigen lokale netwerk.
4. In Home Assistant gebruik je de standaard **Modbus integratie** om verbinding te maken met de Elfin of ESP32.
5. In je automatisering stuur je commando's naar het `Active Power Limit` register van de omvormer.

**Voordeel:** 100% lokaal, geen vreemde Chinese firmware op je netwerk, super betrouwbaar (vooral bij de Ethernet variant).
**Nadeel:** Vereist wat uitzoekwerk m.b.t. de pin-out (welke pinnen op de omvormer zijn A/B voor RS485 of TX/RX voor RS232) en je moet zelf de Modbus-registers opzoeken of uittesten.

---

## Optie 3: Een "Slimme Meter" nabootsen met een ESP32 (Export Limit)
Veel moderne JFY omvormers ondersteunen out-of-the-box een "Export Limit" of "Zero Export" functie. Normaal gesproken koop je daarvoor een speciale energiemeter (zoals een Eastron SDM120 of SDM630) die je in je meterkast hangt en via een lange RS485-kabel verbindt met de omvormer. De omvormer knijpt zichzelf dan automatisch op basis van wat die meter meet.

Aangezien jij de Youless al hebt en geen lange kabel wilt trekken (of geen dure extra meter wilt kopen), kun je deze meter in software "faken".

**Hoe dit werkt:**
1. Je gebruikt een goedkope microcontroller (bijv. een ESP8266 of ESP32) voorzien van **ESPHome** en een klein RS485-bordje.
2. Deze ESP32 leest via de Home Assistant API (of direct via de lokale API van de Youless) continu de huidige P1-waarde (jouw teruglevering) uit.
3. De ESP32 vertaalt deze waarde en stuurt deze via de RS485 kabel naar de omvormer, waarbij hij zich "voordoet" als de originele Eastron of Chint meter die de omvormer verwacht.
4. Je stelt de omvormer in via zijn eigen menu op "Export Limit: 0W". De ingebouwde PID-regeling van de omvormer doet nu zelf al het rekenwerk en knijpt de panelen direct terug.

**Voordeel:** De omvormer doet zelf de razendsnelle regeling (vaak beter en sneller dan een scriptje in Home Assistant).
**Nadeel:** Je moet exact weten welk merk energiemeter de JFY Sunleaf normaal gesproken verwacht voor zijn Export Limit functie.

---

## Hoe nu verder? (Mijn advies)

Aangezien je geen problemen hebt met knutselen en het liefst alles veilig en lokaal houdt:

1. **Test eerst Optie 1 (met internet-blokkade).** Dit is de makkelijkste eerste stap. Blokkeer het IP van je omvormer in je router. Kijk of je de integratie "Solarman" via HACS in Home Assistant kunt installeren. Probeer verbinding te maken met het IP-adres van de omvormer op poort 8899. Als dit lukt en je de waarden binnenkrijgt, ben je al halverwege en hoef je alleen nog maar te experimenteren met welk Modbus-register het vermogen knijpt.
2. **Als dat niet werkt (of niet veilig genoeg voelt):** Ga voor **Optie 2**. Koop een Elfin EW11 of bouw een ESP32 (met ESPHome) met een RS485-module. Prik deze in de omvormer en bestuur de omvormer volledig transparant en onafhankelijk van elke cloud.
3. **De regel-logica:** Zodra je in staat bent om een commando (bijv. 50% vermogen) naar de omvormer te sturen, kun je in Home Assistant een PID-regelaar configureren (hier is ook een HACS integratie voor: `PID controller`) die de `sensor.power_delivered` van de Youless op of rond de `0W` probeert te houden door continu het vermogenspercentage van de omvormer bij te stellen.

Laat me gerust weten of je in één van deze specifieke opties wilt duiken, dan zoeken we de bijbehorende documentatie, componenten of scripts erbij!