# Hvor finder jeg XpressNet protokollen ?


## hvor finder jeg XpressNet protokollen

Du finder den officielle dokumentation og færdige kodebiblioteker til XpressNet-protokollen flere forskellige steder online. Da protokollen er udviklet af Lenz, findes specifikationerne både som rå PDF-filer og som færdige open-source biblioteker til mikrokontrollere som ESP32.

Her er de bedste steder at hente og læse protokollen:

### 1. Det officielle protokol-dokument (PDF)

Lenz har frigivet hele den tekniske specifikation for, hvordan data sendes frem og tilbage på bussen.

* Du kan downloade en ældre, men meget detaljeret udgave af protokolspecifikationen direkte via Rocrail Wiki's XpressNet PDF.
* Du kan også finde en opdateret tysk version af protokollen i manualen til Lenz Interface LAN/USB (Art. nr. 23151), som beskriver alle computer-kommandoerne.

### 2. Færdige kildekoder til ESP32 (Anbefales!)

I stedet for at skrive koden helt fra bunden baseret på det rå PDF-dokument, bør du bruge et færdigt open-source bibliotek. Der findes fremragende Arduino/ESP32-biblioteker, som allerede har tygget sig igennem XpressNet-protokollen:

* Philipp Gahtow's XpressNet-biblioteker: Han er eksperten på området og har lavet biblioteker til både Master- og Slave-funktioner.
  * Hvis din ESP32 skal lytte som en "håndterminal" (enhed på bussen), skal du bruge [Arduino XpressNet Slave på GitHub](https://github.com/Digital-MoBa/XpressNet).
  * Du kan finde yderligere dokumentation på hans dedikerede hjemmeside [pgahtow.de](https://pgahtow.de/w/XpressNet).

### Hvordan ser "Modeltid" ud i protokollen?

Når du kigger i dokumentationen, vil du se, at Lenz-kommandostationen udsender tidsdata i pakker. En typisk tids-pakke på bussen (ofte under "System-status" eller "Modeltog-tid") består af 3-4 bytes:

1. Header-byte: Fortæller, at dette er en tidspakke.
2. Byte 1: Indeholder den aktuelle time (0-23) + ugedag (hvis understøttet).
3. Byte 2: Indeholder det aktuelle minut (0-59).
4. Byte 3: Indeholder den hastighedsfaktor, som uret kører med (f.eks. om modeltiden skal løbe 4 gange hurtigere end normal tid).
For at bruge disse biblioteker i ESPHome skal du inkludere dem som et custom_component via din yaml-fil.

Vil du have et specifikt eksempel på, hvordan du inkluderer Philipp Gahtow's XpressNet-bibliotek direkte som en custom_component i din ESPHome-kode, så du slipper for selv at dekode de rå bytes?

