# Loko på en strækning

## kan jeg se hvilket loko som er på en given strækning

Ja, det kan du godt, men det kræver, at dit modelbaneanlæg har installeret tilbagemeldingsmoduler med RailCom-understøttelse (såsom Lenz LRC120, Lenz LB101 / LB050 kombineret med en RailCom-detektor, eller nyere moduler fra f.eks. Yamorc eller Digikeijs).

Selve XpressNet-bussen og det bibliotek, du bruger, kan godt overføre disse data, men der er en vigtig teknisk forskel på almindelig sporbesatmelder (er der et tog?) og RailCom (hvilket tog er det?).

Her er, hvordan det fungerer i praksis, og hvordan du bringer det ind i din ESP32:

### Hvordan RailCom-data sendes over XpressNet

Når et lokomotiv med en RailCom-dekoder holder på en isoleret strækning med en RailCom-detektor, sker der følgende:

1. Dekoderen i lokomotivet sender sin DCC-adresse (f.eks. Adresse 44) tilbage i skinnerne.
2. RailCom-detektoren fanger dette og sender en besked til din Lenz-central.
3. Lenz-centralen sender en specifik XpressNet-pakke ud på bussen (en udvidet feedback-meddelelse).

### Sådan fanger du det i koden via biblioteket

I XPressNet-biblioteket skal du bruge en callback-funktion (en funktion, der automatisk udløses, når der kommer RailCom-data på bussen).

I din lenz_clock.h fil kan du tilføje funktionen notifyXNetFeedbackRailcom. Her er et principielt C++ eksempel på, hvordan du fanger, hvilket lokomotiv der er på en bestemt strækning (modul- og port-adresse):

```cpp
// Denne funktion kaldes automatisk af XPressNet-biblioteket, 
// når en RailCom-adresse meldes på en strækning.
void notifyXNetFeedbackRailcom(uint8_t UserIndex, uint16_t Address, uint8_t Port, uint16_t LocoAddress) {
  // UserIndex: Bruges hvis du har flere XpressNet-interfaces (ofte 0)
  // Address: Detektorens moduladresse (f.eks. modul 1)
  // Port: Den specifikke strækning/port på modulet (f.eks. spor 1, 2, 3 eller 4)
  // LocoAddress: Lokomotivets DCC-adresse (f.eks. 44)

  if (Address == 1 && Port == 0) {
    // Hvis det er modul 1, port 0 (Strækning A):
    if (LocoAddress > 0) {
      // Formater til f.eks. "Loko 44" og send til en ESPHome text_sensor
      char buffer[20];
      sprintf(buffer, "Loko %d", LocoAddress);
      id(straekning_a_sensor).publish_state(buffer);
    } else {
      // Hvis LocoAddress er 0, betyder det, at strækningen er blevet tom
      id(straekning_a_sensor).publish_state("Tom");
    }
  }
}
```

### Hvad kræver det af din hardware?

For at dette overhovedet vil virke i din ESPHome-opsætning, skal tre ting være opfyldt:

1. I lokomotivet: Dekoderen skal have RailCom slået til (typisk i CV29).
2. På skinnerne: Strækningen skal være isoleret og forbundet gennem et modul, der kan læse RailCom-data (f.eks. tilkoblet Lenz XpressNet via feedback-bussen).
3. I din Lenz-central: Global RailCom-udskæring (RailCom cutout) skal være aktiveret på din central (f.eks. LZV200), så dekoderne har "hul igennem" til at sende data.

Hvis du har disse moduler på dit anlæg i dag, kan du lave dynamiske sensorer i Home Assistant, der ændrer sig fra f.eks. "Tom" til "Loko 3001", alt efter hvilket tog du kører ind på strækningen!

Har du allerede RailCom-kompatible besatmeldere installeret på dine strækninger, eller bruger du i dag almindelige strøm-besatmeldere (uden adresse-id)?

## hvis hvordan jeg laver en git clone af XPressNet kan jeg så includerer koden lokalt

Ja, det kan du sagtens. Det er faktisk en rigtig god idé at lægge koden lokalt, hvis du vil være sikker på, at dit projekt altid kan kompileres, selvom du ikke har internetforbindelse, eller hvis ejeren af GitHub-arkivet skulle ændre i koden.

Når du gør det lokalt i ESPHome, skal du blot placere biblioteket i en undermappe og henvise til det i din .yaml-fil.

Her er den præcise fremgangsmåde:

### 1. Strukturér dine filer

I den mappe, hvor din ESPHome-opsætning ligger (typisk din Home Assistant config/esphome/ mappe), skal du oprette en undermappe til dine eksterne biblioteker.
Kør din git clone så din mappestruktur ser således ud:

```text
config/
└── esphome/
    ├── din_lenz_styring.yaml   <-- Din primære ESPHome konfigurationsfil
    ├── lenz_clock.h            <-- Din tilpassede C++ kode
    └── my_libraries/           <-- Opret denne mappe
        └── XPressNet/          <-- Her lander filerne fra dit Git Clone
            ├── src/
            │   ├── XPressNet.cpp
            │   └── XPressNet.h
            └── library.json
```

### 2. Tilpas din ESPHome YAML-fil

Nu skal du ændre i din .yaml fil under punktet libraries:. I stedet for GitHub-linket, fortæller du nu ESPHome, at den skal lede i din lokale mappe:

```yaml
esphome:
  name: lenz-xpressnet-clock
  
  libraries:
    # Fortæller ESPHome at biblioteket ligger lokalt i undermappen
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino

# ... resten af din konfiguration (wifi, api, text_sensor osv.) er uændret
```

### Hvorfor er den lokale løsning en fordel?

* Ingen internet-afhængighed: ESPHome behøver ikke at hente koden fra GitHub hver gang du installerer eller opdaterer din ESP32.
* Egne modifikationer: Hvis du får brug for at rette direkte i XPressNet.cpp eller XPressNet.h (f.eks. for at fintune timingen eller tilføje specielle Lenz-kommandoer), kan du gøre det med det samme, og ESPHome vil bruge dine rettelser ved næste kompilering.

Har du adgang til at køre git clone direkte i din Home Assistant-mappe (f.eks. via Terminal & SSH add-on), eller overfører du filerne manuelt via Samba/SFTP?

## jeg koder fra visual studio

Når du koder direkte i Visual Studio Code (typisk med ESPHome-udvidelsen eller PlatformIO), er det super nemt at køre en git clone lokalt på din maskine.

Fremgangsmåden er stort set den samme, men du kan gøre det hele direkte via den indbyggede terminal i VS Code.

### 1. Kør Git Clone i VS Code terminalen

1. Åbn din ESPHome-projektmappe i Visual Studio Code.
2. Åbn terminalen i VS Code (`Ctrl + `` eller via menuen Terminal -> New Terminal).
3. Opret mappen til biblioteker og clone projektet ved at køre disse to kommandoer:

```bash
mkdir -p my_libraries
git clone https://github.com my_libraries/XPressNet
```

### 2. Opsætning i din YAML

Da du koder lokalt, vil VS Code og ESPHome lede efter stien relativt til din .yaml-fil. Din konfiguration skal se således ud:

```yaml
esphome:
  name: lenz-xpressnet-clock
  
  libraries:
    # file:// angiver at stien er lokal i forhold til denne yaml-fil
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino

esphome_core_config:
  includes:
    - lenz_clock.h
```

### 3. Få IntelliSense (autocompletion) til at virke i VS Code

Når du arbejder i lenz_clock.h, vil VS Code i starten nok brokke sig med en rød streg under #include <XPressNet.h>, fordi editoren ikke automatisk ved, hvor biblioteket ligger.

Det kan du fikse ved at tilføje stien til din lokale VS Code konfiguration:

1. Tryk Ctrl + Shift + P (eller Cmd + Shift + P på Mac).
2. Vælg "C/C++: Edit Configurations (UI)".
3. Scroll ned til Include path og tilføj en ny linje: ${workspaceFolder}/my_libraries/XPressNet/src (eller bare stien direkte til mappen).

Så forsvinder de røde fejlstreger, og du får fuld fuldførelse (IntelliSense) på alle Lenz- og XpressNet-funktionerne, mens du skriver din C++ kode.

Bruger du ESPHome dashboardet i en browser til at udrulle koden trådløst (OTA) til din ESP32, eller udruller du koden direkte via et USB-kabel fra VS Code?

## Du sagde: platform io implemterit i home assiustant

Når du bruger ESPHome via Home Assistant (f.eks. gennem ESPHome-add-on'en), kører ESPHome faktisk PlatformIO i baggrunden inde i Home Assistant-containeren for at downloade biblioteker og kompilere koden [github.com].

Da du koder lokalt fra Visual Studio Code, har du to geniale måder at strukturere det på, alt efter om du vil have Home Assistant til at stå for kompileringen, eller om du vil køre PlatformIO 100% lokalt på din egen computer.

Her er de to bedste metoder:

### Metode 1: Fortsæt med ESPHome-strukturen (Anbefalet og nemmest)

Selvom Home Assistant bruger PlatformIO i baggrunden, behøver du ikke at konfigurere en traditionel platformio.ini-fil. Du lader bare ESPHome styre PlatformIO for dig via din .yaml-fil.

1. Du gemmer din .yaml, lenz_clock.h og din my_libraries/XPressNet-mappe i din Home Assistant config/esphome/ mappe via VS Code (f.eks. ved hjælp af Samba, SFTP eller VS Code Remote SSH add-on'en).
2. Når du trykker på Install inde i Home Assistant ESPHome-dashboardet, pakker ESPHome automatisk dine lokale filer, fodrer dem til sit interne PlatformIO, kompilerer det hele og sender det trådløst (OTA) ud til din ESP32.

Dette holder din kode fuldstændig synkroniseret med dit Home Assistant-system.

### Metode 2: Kør PlatformIO lokalt på din pc (Hvis du vil udrulle med USB-kabel)

Hvis du har installeret PlatformIO-udvidelsen direkte i din VS Code på computeren, og du gerne vil have fuld kontrol over kompileringen lokalt (og f.eks. udrulle til din ESP32 via USB-kabel før du monterer den på modelbanen), kan du opsætte en ægte PlatformIO-struktur.
Her skal du bruge en platformio.ini-fil i din VS Code-mappe:

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

# Fortæl PlatformIO, hvor dit lokale XPressNet-bibliotek ligger
lib_deps =
    file://my_libraries/XPressNet
```

I denne struktur vil du typisk skrive dit program som en ren Arduino-skitse (main.cpp), der bruger XpressNet-biblioteket, og derefter bruge f.eks. MQTT eller Home Assistant Native API-biblioteket (Native API for Arduino) til at sende data ind til Home Assistant.

#### Hvilken metode skal du vælge?

Hvis målet er at have en ren og simpel integration med Home Assistant, er Metode 1 absolut den bedste. Du bruger VS Code som en avanceret teksteditor, men lader Home Assistants indbyggede ESPHome/PlatformIO-motor gøre det tunge arbejde med at bygge og udrulle firmwaren.

Redigerer du dine filer i VS Code via en netværksforbindelse direkte ind i dit Home Assistant-system (f.eks. via Samba eller SSH), eller har du filerne liggende på din lokale harddisk?
AI-svar kan indeholde fejl. Få flere oplysninger

## Du sagde: den første

Når du redigerer filerne via en direkte netværksforbindelse til dit Home Assistant-system (f.eks. via Samba eller VS Code SSH), er det den absolut mest strømlinede måde at gøre det på. Dit Home Assistant-system og dets indbyggede PlatformIO/ESPHome-motor klarer alt arbejdet, mens du blot bruger VS Code som en lækker editor [github.com].

Her er den præcise tjekliste til, hvordan du udfører det i din VS Code, så det spiller 100% sammen med Home Assistant:

### 1. Placer filerne rigtigt via dit netværksdrev

Når du kigger i din VS Code (som er forbundet til din Home Assistant), skal du sikre dig, at dine filer ligger i config/esphome/ mappen i dette præcise layout:

```text
config/
└── esphome/
    ├── lenz_styring.yaml        <-- Din ESPHome konfigurationsfil
    ├── lenz_clock.h             <-- Din C++ fil (med XPressNet-koden)
    └── my_libraries/
        └── XPressNet/           <-- Heri ligger de Git-clonede filer (src/, library.json osv.)
```

### 2. Din endelige lenz_styring.yaml

Sørg for, at din YAML-fil i VS Code indeholder følgende linjer i toppen, så ESPHhomes interne PlatformIO-motor ved, at den skal snappe det lokale bibliotek med i kompileringen:

```yaml
esphome:
  name: lenz-xpressnet-clock
  libraries:
    - file://my_libraries/XPressNet

esp32:
  board: esp32dev
  framework:
    type: arduino  # Meget vigtigt, da XPressNet-biblioteket er skrevet til Arduino

esphome_core_config:
  includes:
    - lenz_clock.h
```

### 3. Sådan udruller du koden

Når du har gemt dine filer i VS Code, behøver du ikke gøre mere på din computer:

1. Gå ind i din browser og åben dit Home Assistant Dashboard.
2. Klik på ESPHome i sidemenuen.
3. Find din lenz-xpressnet-clock på listen (den vil dukke op automatisk, fordi filen ligger i mappen).
4. Klik på de tre prikker og vælg Install -> Wirelessly (hvis din ESP32 allerede er på nettet) eller vælg den metode, der passer dig.

Nu vil Home Assistant starte sin interne PlatformIO-kompiler, læse dit lokale XPressNet-bibliotek, bage det hele sammen med din lenz_clock.h og sende det direkte ud til din ESP32 på modelbanen [github.com].

Har du allerede loddet eller forbundet dit RS485-modul til din ESP32, eller vil du have, at vi kigger på de præcise GPIO-pins, inden du flasher koden første gang?

## Du sagde: du må gerne foreslå forbindelser

Her er det mest optimale og stabile diagram til at forbinde din ESP32 med et typisk MAX485 / RS485 til TTL-modul, så det kan snakke med Lenz XpressNet-bussen.

Da XpressNet kører med en ret høj hastighed (62.500 baud), er det vigtigt at bruge ESP32'erens dedikerede hardware-serieport (UART2). Det sikrer, at der ikke tabes data eller pakker fra uret.

### 1. Ledningsføring: ESP32 til RS485-modul

|ESP32 Pin|RS485 Modul Pin|Forklaring|
|:---|:---|:---|
|5V eller 3V3|VCC|Strøm (tjek om dit modul kræver 5V eller 3.3V)|
|GND|GND|Fælles stel|
|GPIO 16 (RX2)|RXD / RO|Modtagelse af data fra bussen|
|GPIO 17 (TX2)|TXD / DI|Afsendelse af data til bussen|
|GPIO 4|DE og RE (Luse sammen)|Direction Control (Skifter mellem lytte/sende)*|

* ***Bemærk: Hvis du har købt et "automatisk" RS485-modul (Hardware Flow Control), har det typisk ikke DE/RE-pins. Så skal du bare helt udelade GPIO 4.***

### 2. Ledningsføring: RS485-modul til Lenz XpressNet (Skinner/Central)

XpressNet bruger typisk enten et RJ12-stik (som i et telefonkabel) eller et 5-polet DIN-stik (afhængigt af dine paneler eller central). Du skal kun bruge 3 ledninger herfra ind til dit RS485-modul:

|Lenz XpressNet Signal|RS485 Modul Pin|RJ12 Pin-nummer|DIN 5-pol Pin-nummer|
|:---|:---|:---|:---|
|RS485 B (Data -)|B / B-|Pin 3|Pin 5|
|RS485 A (Data +)|A / A+|Pin 4|Pin 3|
|M (GND / Ground)|GND|Pin 2|Pin 4|

* **Vigtigt: XpressNet-kablet fører også 12V strøm** (Pin 1 og 6 i RJ12 / Pin 1 og 2 i DIN). **Lad være med at forbinde 12V direkte til din ESP32**, medmindre du bruger en step-down regulator (f.eks. en LM2596) til at lave det om til 5V! Det sikreste i starten er at give ESP32 strøm via et helt almindeligt USB-kabel.

### 3. Justering i din lenz_clock.h

Hvis du følger ovenstående diagram, passer det perfekt med de definitioner, vi lagde ind i C++ koden i lenz_clock.h tidligere:

```cpp
#define XNET_RX_PIN 16  // Forbundet til RO/RXD
#define XNET_TX_PIN 17  // Forbundet til DI/TXD
#define XNET_DE_PIN 4   // Forbundet til DE/RE (hvis dit modul har dem)
```

Når du har forbundet det på denne måde, vil din ESP32 være klar til at lytte direkte på XpressNet-linjen i det øjeblik, du tænder for din Lenz-central.

Har du et RS485-modul med DE/RE-pins (ofte de små billige røde moduler), eller har du fået fat i den type med automatisk retningsstyring (ofte lyseblå moduler)?
