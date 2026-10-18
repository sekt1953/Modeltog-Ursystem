# Lenz time 

## hvordan læser jeg tiden fra lenz håndterminal

Måden, du aflæser eller indstiller systemtiden på en Lenz modeljernbane-håndterminal (såsom den udbredte Lenz LH101 koblet til en LZV200-central), gøres via håndkontrollens indbyggede menusystem.
Her er trinene til at finde og aflæse/indstille tiden:

### Trin-for-trin guide (Lenz LH101)

1. Gå til indstillinger: Tryk på tasten M (Menu) for at åbne hovedmenuen.
2. Find systemopsætning: Drej på kontrolknappen (drejeknappen) indtil du ser "Setup System" (eller "Systemeinstellungen" på tysk) i displayet. Tryk på knappen for at vælge.
3. Vælg tidsmenuen: Drej igen på knappen for at navigere frem til menupunktet "Zeit stellen" (eller det tilsvarende engelske/danske menupunkt for ur/tid). Tryk på knappen.
4. Aflæs og indstil: Displayet vil nu vise den aktuelle systemtid samt tidsfaktoren (hvor hurtigt modeltiden skal løbe). Du kan blot aflæse den eller trykke på drejeknappen for at ændre timer og minutter, hvis uret skal synkroniseres.
5. Gem og afslut: Tryk på M for at gemme indstillingerne i din central. Tryk herefter på Esc for at gå helt tilbage til den normale køreskærm.

Bemærk: Hvis du bruger den ældre Lenz LH100-håndterminal, tilgås funktioner oftest via F (Funktion) fulgt af tal-kombinationer (f.eks. Sys eller Prog menuerne) alt efter din centrals softwareversion.

Hvis du ønsker det, kan jeg hjælpe dig med at finde de præcise taster til at ændre tidsfaktoren (modeltiden), eller guide dig i, hvordan du skifter menusproget på din terminal?

## jeg vil gerne hente denne tid ud til min home assistant via en esp32 som er kodet esphome

At hente systemtiden ud fra din Lenz-central (via XpressNet-bussen) til Home Assistant ved hjælp af en ESP32 med ESPHome kræver et par specifikke hardware- og softwaretrin.

Lenz anvender XpressNet, som elektrisk set kører på en RS485-bus-standard. ESP32 kan ikke snakke direkte med RS485 uden et mellemliggende modul.
Her er den mest direkte og stabile måde at bygge det på:

### 1. Nødvendig Hardware

* ESP32 Development Board (f.eks. NodeMCU ESP32).
* RS485 til TTL (UART) konverter-modul (f.eks. et baseret på MAX485 eller de automatiske "hardware-flow-control" moduler).
* Et kabel/stik (typisk RJ12 eller DIN-stik afhængigt af din Lenz-opsætning) til at forbinde til XpressNet.

#### Ledningsføring (XpressNet -> RS485 -> ESP32):

1. Fra XpressNet: Forbind A og B (XpressNet datalinjer) til A og B på RS485-modulet.
2. Fra RS485 til ESP32:
   * VCC -> 5V eller 3.3V (afhængigt af dit modul)
   * GND -> GND
   * RXD -> en ledig GPIO på ESP32 (f.eks. GPIO16)
   * TXD -> en ledig GPIO på ESP32 (f.eks. GPIO17)

### 2. Software & ESPHome Kode

Da ESPHome ikke har en "indbygget" færdiglavet komponent til Lenz XpressNet-ur, skal du bruge ESPHomes uart-komponent og skrive en lille smule tilpasset C++ kode (custom_component eller lambda) for at lytte efter XpressNet-tidspakken.

Lenz-centralen udsender regelmæssigt en pakke på XpressNet med systemtiden (tid og tidsfaktor). Pakkeformatet til tid i XpressNet starter typisk med en specifik header-byte, hvorefter timer og minutter følger.

Her er et eksempel på, hvordan din ESPHome YAML-konfiguration kan se ud. Den opsætter en UART-forbindelse og opretter et tekst-sensor-element (text_sensor), som opdateres i Home Assistant, hver gang der modtages data:

```yaml
esphome:
  name: lenz-xpressnet-bridge

esp32:
  board: esp32dev

# Forbindelse til Home Assistant
api:

wifi:
  ssid: "Dit_WiFi_Navn"
  password: "Dit_WiFi_Kodeord"

# Opsætning af den fysiske kommunikation med RS485 modulet
uart:
  id: xpressnet_bus
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 62500  # XpressNet kører specifikt 62.500 baud

text_sensor:
  - platform: template
    name: "Lenz Systemtid"
    id: lenz_time
    icon: "mdi:clock"

# Logik der lytter på rå bytes fra centralen
custom_component:
  - lambda: |-
      auto my_custom_sensor = new UARTDebugComponent(id(xpressnet_bus));
      return {my_custom_sensor};

# Alternativt kan du fange data direkte i en loop/lambda i ESPHome:
interval:
  - interval: 1s
    then:
      - lambda: |-
          // Eksempel på simpel byte-læsning. 
          // XpressNet tidspakker har typisk formatet: 0x83 0x00 (Timer) (Minutter) (XOR-Checksum)
          while (id(xpressnet_bus).available()) {
            uint8_t c = id(xpressnet_bus).read();
            // Her indsættes en tilstandsmaskine (State Machine) der leder efter 0x83
            // Når pakken er valideret, kan tiden pushes til Home Assistant:
            // id(lenz_time).publish_state("12:34");
          }
```

### 3. Den præcise XpressNet protokol (Til din kode)

For at få lambda-koden helt skarp, skal du kigge efter XpressNet pakken for "Time and speed multiplier".

* Header-byte: 0x83
* Data-byte 1: 0x00
* Data-byte 2: 0xTT (Timer, kodet i 24-timers format binary/BCD)
* Data-byte 3: 0xMM (Minutter)
* Data-byte 4: 0xEE (Ekstra byte til ugedag/tidsfaktor)
* Data-byte 5: Checksum (XOR af de foregående bytes)

#### Den nemme genvej: Brug et færdigt bibliotek

Hvis du ikke vil skrive rå byte-parsing i din YAML, kan du trække det populære Arduino-bibliotek XpressNetMaster / XpressNetClient ind som et eksternt bibliotek i din ESPHome-kode under libraries: feltet. Dette bibliotek har færdige funktioner som getXpressNetClock(), der spytter timer og minutter direkte ud, som du derefter let kan sende videre op til Home Assistant via id(lenz_time).publish_state().

## kan du vise koden med xpressnetmaster includeret ?

Her er et komplet eksempel på, hvordan du sætter din ESPHome YAML-konfiguration op, hvor Arduino-biblioteket XPressNet (oprindeligt udviklet af Philipp Gahtow) inkluderes automatisk, og tiden hentes ud via en tilpasset C++ sensor [github.com].

Da ESPHome kompilerer koden som C++, kan vi inkludere biblioteket og lave en lille klasse (LenzClockSensor), der lytter på XpressNet-bussen og opdaterer en text_sensor i Home Assistant, hver gang uret ændrer sig.

### 1. Hardware-forudsætning for koden

XpressNet kører med en speciel hastighed på 62500 baud, og da det er en RS485-bus, skal dit RS485-modul bruge en pin til at skifte mellem at sende og modtage (Direction Pin / MAX485_DE_RE).

### 2. ESPHome YAML Konfiguration

Du kan indsætte følgende kode direkte i din ESPHome .yaml fil. Husk at tilpasse dine Wi-Fi oplysninger og GPIO-pins, så de matcher din ESP32.

```yaml
esphome:
  name: lenz-xpressnet-clock
  # Her fortæller vi ESPHome, at den skal downloade XPressNet-biblioteket fra GitHub
  libraries:
    - https://github.com

esp32:
  board: esp32dev
  framework:
    type: arduino  # Biblioteket kræver Arduino-frameworket

wifi:
  ssid: "Dit_WiFi_Navn"
  password: "Dit_WiFi_Kodeord"

api: # Gør sensoren tilgængelig i Home Assistant

logger:
  # Vi dæmper loggeren en smule på UART, så den ikke forstyrrer XpressNet data
  level: DEBUG

# Opsætning af den tilpassede C++ kode direkte i YAML
esphome_core_config:
  includes:
    - lenz_clock.h

text_sensor:
  - platform: custom
    lambda: |-
      auto xnet_clock = new LenzClockSensor();
      App.register_component(xnet_clock);
      return {xnet_clock->time_sensor};
```

### 3. Opret filen lenz_clock.h

For at holde koden ren, skal du oprette en fil i samme mappe som din ESPHome-konfiguration og kalde den lenz_clock.h. Indsæt følgende C++ kode i den:

```cpp
#include "esphome.h"
#include <XPressNet.h>

// Definer de GPIO pins du har forbundet til dit RS485 modul
#define XNET_RX_PIN 16
#define XNET_TX_PIN 17
#define XNET_DE_PIN 4   // Direction pin (DE/RE) på dit MAX485-modul

class LenzClockSensor : public esphome::Component {
 public:
  // Vi opretter en text_sensor, som ESPHome sender til Home Assistant
  esphome::text_sensor::TextSensor *time_sensor = new esphome::text_sensor::TextSensor();
  
  XPressNet xnet;
  int last_minute = -1;
  int last_hour = -1;

  void setup() override {
    // Initialiser XPressNet biblioteket på ESP32 Hardware Serial 2
    // Parametre: Serial-port, RX-pin, TX-pin, Direction-pin
    xnet.setup(2, XNET_RX_PIN, XNET_TX_PIN, XNET_DE_PIN);
    
    time_sensor->set_name("Lenz Systemtid");
  }

  void loop() override {
    // Lad biblioteket håndtere indkommende data på bussen
    xnet.update();

    // Hent data ud fra bibliotekets interne tids-variable
    // XPressNet biblioteket opdaterer automatisk disse ved modtagelse af 0x83 pakker
    int current_hour = xnet.getHour();
    int current_minute = xnet.getMinute();

    // Hvis tiden har ændret sig, sender vi den til Home Assistant
    if (current_minute != last_minute || current_hour != last_hour) {
      if (current_hour >= 0 && current_hour < 24 && current_minute >= 0 && current_minute < 60) {
        last_hour = current_hour;
        last_minute = current_minute;
        
        // Formater tiden som f.eks. "14:05"
        char time_string[6];
        sprintf(time_string, "%02d:%02d", current_hour, current_minute);
        
        // Udgiv staten så Home Assistant modtager den med det samme
        time_sensor->publish_state(time_string);
      }
    }
  }
};
```

### Sådan virker det i praksis:

1. XPressNet.h lytter i baggrunden på Hardware Serial 2 i din ESP32 med de korrekte 62500 baud, som Lenz-centralen kræver.
2. Hver gang uret tikker på din håndterminal (eller centralen udsender sin statuspakke), fanger biblioteket 0x83 (Time/Speed multiplier) pakken automatisk [github.com].
3. loop()-funktionen i koden tjekker konstant, om timer eller minutter har ændret sig. Så snart der er en ændring, formateres det til en klassisk tidsstreng (f.eks. 12:30) og sendes direkte op til din Home Assistant via det indbyggede ESPHome API.

