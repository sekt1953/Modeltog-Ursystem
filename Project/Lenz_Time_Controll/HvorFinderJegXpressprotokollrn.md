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

## hvis jeg vil sætte tiden i Lenz med tid fra Home Assistant skal jeg så bruge maste bivlioteket

Ja, hvis du vil sætte tiden i dit Lenz-system (altså overskrive eller opdatere tiden i kommandostationen) med tiden fra Home Assistant, skal din ESP32 kunne sende kommandoer på bussen på en måde, der påvirker systemet.

Dog skal du ikke bruge Master-biblioteket, men derimod stadig **Slave-biblioteket (XpressNet-Slave)**.

Her er forklaringen på hvorfor, og hvordan det fungerer:

### Hvorfor Slave-biblioteket er det rigtige

I et Lenz-system er det **altid din Lenz-kommandostation (f.eks. LZV100/LZV200), der er Master** på XpressNet-bussen. Der må kun være én Master på bussen af gangen.

* **Master-biblioteket:** Bruges kun, hvis du vil bygge din egen centralstation fra bunden (hvor ESP32 erstatter Lenz-boksen).
* **Slave-biblioteket:** Bruges, når din ESP32 skal opføre sig som et tilbehør på bussen – fuldstændig ligesom en ekstra håndterminal (f.eks. en LH101) eller et pc-interface.

Ligesom du kan indstille modeltiden på dit Lenz-anlæg ved at trykke på knapperne på en fysisk LH101-håndterminal, kan din ESP32 (som en Slave-enhed) sende den nøjagtige samme "Set Model Time"-kommando afsted på bussen.

### Sådan fungerer dataflowet

1. **Home Assistant** sender den aktuelle tid (eller modeltid) til din ESP32 via Wi-Fi (ESPHome synkroniseringen).
2. Din ESP32 modtager tiden og pakker den om til en XpressNet-kommando via **Slave-biblioteket**.
3. ESP32 sender pakken ud på RS485-bussen.
4. Lenz-centralen modtager kommandoen, opdaterer sit interne ur, og broadcaster derefter den nye tid ud til alle dine andre håndterminaler.

### Protokol-kommandoen (Teknisk info)

I XpressNet-protokollen (v3) hedder den kommando, du skal sende, typisk **"Set Model Time"**.
I Philipp Gahtows XpressNet-bibliotek er der ofte en færdig funktion til dette, hvor du blot kalder noget i retning af:

```cpp
// Pseudokode i dit ESPHome custom component:
XpressNet.setClock(timer, minutter, hastighedsfaktor);
```

Dette vil generere den korrekte XpressNet-byte-sekvens (som typisk starter med instruktions-byten 0x21 efterfulgt af data-bytes) og sende den sikkert afsted, når det er ESP32'erens tur til at tale på bussen.

Vil du have hjælp til at strukturere den custom_component C++ kode i ESPHome, der lytter efter tidsændringer fra Home Assistant og skyder dem afsted på XpressNet-bussen?

## Ja tak

Her er den komplette guide og kildekode til at bygge en ESPHome Custom Component, som modtager tiden fra Home Assistant og sender den videre til dit Lenz-system via XpressNet-bussen.

### 1. Hardware-forudsætning

Sørg for, at din ESP32 er forbundet til dit **RS485-til-TTL-modul** via hardware UART-pins (i dette eksempel bruger vi **GPIO16 som RX** og **GPIO17 som TX)**. Husk, at XpressNet altid kører på præcis **62500 baud**.

### 2. Opret C++ filen (xpressnet_clock.h)

Du skal oprette en ekstern C++ fil, som håndterer XpressNet-protokollen. Placer denne fil i din ESPHome-mappe (samme sted som din .yaml-fil). Filen skal hedde **xpressnet_clock.h**.

Koden lytter efter ændringer i timer og minutter, og når tiden skifter, pakker den en officiel XpressNet v3 **"Set Model Time"** kommando (0x21 0x21) og sender den ud på bussen.

```cpp
#include "esphome.h"

class XpressNetClock : public Component, public UARTDevice {
 public:
  XpressNetClock(UARTComponent *parent) : UARTDevice(parent) {}

  void setup() override {
    // Komponent-initialisering hvis nødvendigt
  }

  // Funktion der kaldes fra ESPHome YAML, når tiden ændres
  void send_time_to_lenz(int hours, int minutes, int speed_factor) {
    // XpressNet v3 kommando til at sætte modeltid:
    // Byte 1: 0x21 (Kommando-gruppe)
    // Byte 2: 0x21 (Under-kommando: Sæt ur)
    // Byte 3: Base ugedag (0 = Mandag) + Timer (0-23)
    // Byte 4: Minutter (0-59)
    // Byte 5: Hastighedsfaktor (1-255, hvor 1 er realtid, 0 stopper uret)
    // Byte 6: XOR XOR-Checkbyte (Sikkerhedstjek af de forrige bytes)

    uint8_t byte3 = hours & 0x1F; // Vi antager standard ugedag 0 (00000000 | hours)
    uint8_t byte4 = minutes & 0x3F;
    uint8_t byte5 = speed_factor & 0xFF;

    // Beregn XOR kontrolsum (XpressNet standard)
    uint8_t xor_checksum = 0x21 ^ 0x21 ^ byte3 ^ byte4 ^ byte5;

    // Send pakken til RS485 / XpressNet bussen
    this->write_byte(0x21);
    this->write_byte(0x21);
    this->write_byte(byte3);
    this->write_byte(byte4);
    this->write_byte(byte5);
    this->write_byte(xor_checksum);
    this->flush();

    ESP_LOGD("custom_xpressnet", "Sendte tid til Lenz: %02d:%02d (Hastighed: 1:%d)", hours, minutes, speed_factor);
  }
};
```

### 3. ESPHome-konfigurationen (din_esp32.yaml)

I din primære ESPHome-konfiguration skal du nu inkludere din nye C++ fil, opsætte UART-bussen, hente tiden fra Home Assistant og lave en automatisering, der sender tiden videre **hvert minut**.

```yaml
esphome:
  name: esp32-lenz-clock
  # Inkluder din custom C++ kode her
  includes:
    - xpressnet_clock.h

esp32:
  board: esp32dev
  framework:
    type: arduino

# Opsætning af RS485 UART forbindelsen til XpressNet
uart:
  id: xpressnet_uart
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 62500

# Registrer vores Custom Component i ESPHome
custom_component:
  - id: my_xpressnet_clock
    lambda: |-
      auto my_clock = new XpressNetClock(id(xpressnet_uart));
      return {my_clock};

# Hent den officielle tid fra Home Assistant
time:
  - platform: homeassistant
    id: ha_time
    timezone: "Europe/Copenhagen"
    
    # Hvert minut når uret skifter, sendes den nye tid til Lenz bussen
    on_time:
      - seconds: 0
        then:
          - lambda: |-
              // Hent aktuelle timer og minutter fra ESPHome tidsmodulet
              auto time_now = id(ha_time).now();
              int hours = time_now.hour;
              int minutes = time_now.minute;
              
              // Hastighedsfaktor: 1 betyder normal tid (realtid)
              int speed_factor = 1; 
              
              // Kald vores C++ funktion og skyd data afsted
              static_cast<XpressNetClock*>(id(my_xpressnet_clock)).get_component(0)->send_time_to_lenz(hours, minutes, speed_factor);
```

### Sådan fungerer det i praksis:

1. Din ESP32 starter op og forbinder til dit Wi-Fi og Home Assistant.
2. time: - platform: homeassistant synkroniserer automatisk uret på ESP32 med dansk tid.
3. Hvert minut (når sekunder slår 0), vågner on_time triggeren.
4. Den sender timer, minutter samt en hastighedsfaktor (sat til 1 for realtid) igennem vores C++ funktion.
5. C++ koden beregner den nødvendige XOR-kontrolsum, som Lenz kræver for at acceptere data, og spytter de 6 bytes direkte ud i XpressNet-bussen via dit RS485-modul.
6. Din Lenz-central modtager signalet, og du vil med det samme kunne se din ur-opdatering på skærmen af din Lenz LH101 håndterminal.