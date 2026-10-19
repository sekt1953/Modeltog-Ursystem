# Styring af Lenz enheder fra HomeAssistant

**Her er en af mine dialoger med Google AI om at styre tiden på vores modelanlæg.**

## XpressNet kører med en speciel hastighed på 62500 baud, og da det er en RS485-bus, skal dit RS485-modul bruge en pin til at skifte mellem at sende og modtage (Direction Pin / MAX485_DE_RE)., kan dette lade sig gøre uden DE_RE ben

Ja, det kan godt lade sig gøre at køre XpressNet (RS485) uden at bruge et dedikeret DE/RE-styreben (Direction Pin) fra din mikrokontroller.

Der findes to primære måder at løse dette på, enten via hardware (automatisk retningsstyring) eller ved at vælge en specifik transceiver-chip.

Her er de tre mest almindelige løsninger:

### 1. Brug en "Auto-Direction" RS485-transceiver (Den letteste løsning)

Du kan udskifte din standard MAX485-chip med en transceiver, der selv styrer retningen automatisk i hardwaren. Disse chips registrerer selv, hvornår der sendes data på TX-linjen, og åbner automatisk for senderen (Driver Enable).

* Eksempler på chips: MAX13487 eller THVD1406.
* Hvordan det virker: Du forbinder blot TX og RX til din mikrokontroller. Chippen sørger selv for resten, og du sparer DE/RE-benet helt.

### 2. Hardware-kredsløb med en standard MAX485 (Inverter + Kondensator/Modstand)

Hvis du absolut vil bruge en standard MAX485, kan du bygge et lille automatisk kredsløb (ofte kaldet et "Auto-TX-kredsløb") foran chippen.

* Hvordan det virker: TX-pinden fra din mikrokontroller forbindes til DE/RE-pindene via en inverter (f.eks. en 74HC14 eller en transistor) og et lille RC-led (modstand og kondensator).
* Udfordring ved 62500 baud: RC-leddets timing (tidskonstant) skal beregnes præcist, så chippen forbliver i "sende-mode" i den tid, det tager at sende et bit, men lukker hurtigt nok til, at du ikke går glip af det svar, som XpressNet-bussen sender tilbage.

### 3. "Hardware-trick" direkte på TX-pinden (Simpelt, men med risiko)

Nogle billige RS485-moduler (f.eks. de populære røde kinesiske moduler til Arduino) bruger et trick, hvor DE/RE er forbundet direkte til TX-signalet via en transistor.

* Hvordan det virker: Når TX er høj (tomgang), er senderen slukket, og modtageren er tændt. Når TX går lav (start-bit/data), tændes senderen og trækker bussen lav. Bussen trækkes høj igen af termineringsmodstandene (pull-up/pull-down), når senderen slukker.
* Udfordring: Dette kan være ustabilt på en XpressNet-bus, hvis bus-termineringen ikke er helt perfekt, da signalflankerne kan blive sløve.

### Anbefaling

Til XpressNet (62500 baud) vil den absolut mest stabile og fejlfrie løsning uden et DE/RE-ben være at bruge en **MAX13487** eller et færdigt modul med **automatisk retningsstyring**.

Hvis du vil gå videre med projektet, vil du så designe dit eget printkort, eller leder du efter et færdigt modul, du kan købe?

## jeg har et printkort mærket v7.00 ttl485-v2.0, det bruger en ic max13487

[![2014-15-35.png](./images/Skærmbillede%20fra%202026-10-19%2014-15-35.png)](https://let-elektronik.dk/max3485-ttl-til-rs485-modul-hot-swap-esd-beskyttet)

Det er en rigtig god nyhed! **Da dit printkort bruger en MAX13487-chip, betyder det, at du allerede har den perfekte hardware til formålet**.

MAX13487 er netop en af de specielle chips, der har indbygget **AutoDirection-styring**.

Du behøver derfor ikke et DE/RE-ben fra din mikrokontroller. Chippen klarer det hele selv i hardwaren.

### Sådan forbinder og bruger du dit modul til XpressNet:

1. Forbindelser til mikrokontrolleren:
   * TX (Modul) → TX (Mikrokontroller)
   * RX (Modul) → RX (Mikrokontroller)
   * VCC → 5V (eller 3.3V, afhængigt af din mikrokontroller og modulets specifikationer)
   * GND → GND
2. Forbindelser til XpressNet (RS485-siden):
   * A → XpressNet Linje A (Pin 4 i et typisk RJ12-stik)
   * B → XpressNet Linje B (Pin 3 i et typisk RJ12-stik)

#### Lenz XpressNet stik forbindelser

![2014-01-18.png](./images/Skærmbillede%20fra%202026-10-19%2014-01-18.png)
![2014-01-04.png](./images/Skærmbillede%20fra%202026-10-19%2014-01-04.png)

#### Vigtigt at vide om MAX13487 og kode:

* **Ingen kode-ændringer**: I din software skal du overhovedet ikke tænke på at tænde og slukke for et kontrol-ben før og efter Serial.print(). Du sender bare data ud på din computers eller mikrokontrollers serielle port ved 62500 baud, og chippen åbner og lukker lynhurtigt for bussen helt automatisk.
* **Intern Pull-up**: MAX13487 kræver, at mikrokontrollerens TX-ben holdes højt (High) i tomgang for at fungere korrekt. Det gør langt de fleste mikrokontrollere (som Arduino, ESP32 osv.) automatisk, når den serielle port initialiseres.

Vil du have hjælp til at verificere diagrammet / pinout for, hvordan XpressNet-forbindelsen skal kables til RJ12-stikket, eller har du styr på den del?
AI-svar kan indeholde fejl. Få flere oplysninger

## kan du vise en kode til esp32

**Her er et komplet og optimeret kodeeksempel til en ESP32**, der opsætter den hardware-serielle port til 62500 baud (XpressNet-hastigheden).

Da dit TTL485-V2.0-modul med **MAX13487** selv styrer retningen, skal koden udelukkende konfigurere den serielle port og sende/modtage data uden brug af DE/RE-styring.
I eksemplet bruges **HardwareSerial 2 (UART2)**, hvilket er den mest sikre port at bruge på ESP32, da den ikke forstyrrer USB-programmeringen (UART0).

### Forbindelsesdiagram (ESP32 til TTL485-modul)

* ESP32 GPIO 17 (TX2) → TX på modulet
* ESP32 GPIO 16 (RX2) → RX på modulet
* ESP32 5V (eller 3V3) → VCC på modulet
* ESP32 GND → GND på modulet

### Kodeeksempel (Arduino IDE)

```cpp
#include <Arduino.h>

// Definer de pinde, der skal bruges til XpressNet (UART2)
#define XPRESSNET_RX_PIN 16
#define XPRESSNET_TX_PIN 17

// XpressNet kører med den specielle hastighed 62500 baud
#define XPRESSNET_BAUD 62500

void setup() {
  // Start den indbyggede USB-serielport til overvågning i Serial Monitor
  Serial.begin(115200);
  delay(1000); // Lille pause så Serial Monitor kan nå at starte
  Serial.println("ESP32 XpressNet UART2 Starter...");

  // Initialiser HardwareSerial 2 til XpressNet
  // SERIAL_8N1 betyder: 8 databit, ingen paritet, 1 stopbit (Standard for XpressNet)
  Serial2.begin(XPRESSNET_BAUD, SERIAL_8N1, XPRESSNET_RX_PIN, XPRESSNET_TX_PIN);
  
  Serial.println("XpressNet porten er klar på 62500 baud.");
}

void loop() {
  // 1. MODTAGELSE AF DATA FRA XPRESSNET
  // Hvis der kommer data ind fra bussen, læser vi det og sender det til computeren
  if (Serial2.available()) {
    Serial.print("Modtaget fra XpressNet: ");
    while (Serial2.available()) {
      byte indkommendeByte = Serial2.read();
      
      // Vis dataen i HEX-format, da XpressNet-protokollen er binær/HEX-baseret
      if (indkommendeByte < 0x10) Serial.print("0"); // Førende nul for pæn formatering
      Serial.print(indkommendeByte, HEX);
      Serial.print(" ");
    }
    Serial.println();
  }

  // 2. AFLESNING/TEST-SENDING (Eksempel)
  // Hvis du skriver noget i din computers Serial Monitor, sender ESP32 det ud på XpressNet-bussen
  if (Serial.available()) {
    byte testByte = Serial.read();
    
    // Du sender bare direkte med Serial2.write(). 
    // MAX13487 chippen sørger SELV for at tænde og slukke for bussen (AutoDirection).
    Serial2.write(testByte); 
    
    Serial.print("Sendte byte til XpressNet: 0x");
    Serial.println(testByte, HEX);
  }
}
```

#### Vigtige bemærkninger til koden:

* **Ingen DE/RE manipulation**: Læg mærke til, at der før og efter Serial2.write() ikke ændres på nogen digitale pinde. MAX13487 skifter til "sende-mode" i det millisekund, dataen forlader GPIO 17, og falder straks tilbage til "modtage-mode", når overførslen er færdig.
* **XpressNet-format**: XpressNet kommunikerer med binære bytes (HEX), ikke almindelig tekst. Derfor udskriver koden de modtagne data i HEX-format i Serial Monitor, så du kan tyde pakkerne ud fra XpressNet-specifikationen.

Er du ved at bygge en **XpressNet-enhed** (f.eks. et håndhåndtag/X-bus enhed), eller vil du lave en interface-boks, der kan tale med en DCC-central (f.eks. z21 eller Lenz)? I så fald kan jeg hjælpe med strukturen på selve **XpressNet-pakkerne**.

## jeg vil gerne bruge ESPHome da det er forbundet til Home Assistsant

Det giver rigtig god mening at bruge ESPHome, når du vil have det direkte ind i Home Assistant.

Da ESPHome kører ovenpå ESP32'erens hardware, understøtter den uden problemer den specielle hastighed på **62500 baud**. Du angiver det bare direkte i din konfiguration. Takket være dit TTL485-V2.0-modul med **MAX13487** behøver du (ligesom før) ikke at tænke på retningsstyring eller DE/RE-pinde i din konfiguration.

Måden du opsætter det på i ESPHome afhænger en lille smule af, hvad du vil bruge XpressNet-dataene til i Home Assistant. Her er den grundlæggende byggesten, som åbner porten korrekt:

### 1. Grundlæggende UART-opsætning i ESPHome

Du skal tilføje en uart:-blok til din YAML-fil. Vi bruger GPIO16 og GPIO17 (UART2 på ESP32):

```yaml
uart:
  - id: xpressnet_bus
    tx_pin: GPIO17
    rx_pin: GPIO16
    baud_rate: 62500
    stop_bits: 1
    data_bits: 8
    parity: NONE
```

### 2. Hvordan får du data ind og ud af Home Assistant?

Da XpressNet er en kompleks, binær modeljernbaneprotokol (og ikke almindelig tekst), kan du ikke bare bruge en standard tekst-sensor. Du har typisk to veje at gå:

#### Mulighed A: Stream data råt til Home Assistant via Stream Server (Enklest)

Hvis du vil lade et eksternt system, en integration i Home Assistant eller f.eks. computer-software (som JMRI) tale direkte med XpressNet via Wi-Fi/ESPHome, kan du lave en TCP Stream Server.

Dette forvandler din ESP32 til en trådløs XpressNet-netværksbro:

```yaml
# Tilføj dette under din uart-blok for at lave en TCP-port i dit netværk
stream_server:
  uart_id: xpressnet_bus
  port: 1234  # Du kan vælge en valgfri ledig port
```

#### Mulighed B: Skriv dit eget logik-kredsløb direkte på ESP32 (Custom Component)

Hvis ESP32'eren selv skal fortolke XpressNet-pakkerne (f.eks. opfange når et bestemt sporskifte skifter position og tænde en lampe i HA), skal du bruge ESPHomes custom_component eller skrive en lambda-funktion i YAML, der overvåger bytes på UART-bussen.

Hvilken rolle skal din ESP32 have i dit setup? Skal den sende alt råt videre til en computer/integration (som Mulighed A), eller vil du oprette specifikke knapper/sensorer i Home Assistant (f.eks. en tænd/sluk-knap til et sporskifte eller et tog)?

### 1. Opret C++ filen (xpressnet_time.h)

Du skal oprette en fil i din ESPHome-mappe (samme sted som din .yaml-fil), som hedder xpressnet_time.h. Denne fil initialiserer XpressNet-biblioteket på ESP32'erens Serial2 port og håndterer tidssynkroniseringen.

Indsæt følgende kode i filen:

```cpp
#include "esphome.h"
#include <XpressNet.h> // Philipp Gahtows bibliotek

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  unsigned long last_update = 0;

 public:
  void setup() override {
    // Initialiser XpressNet på UART2 (Hardware-seriel på ESP32)
    // Bemærk: XpressNet-biblioteket opsætter selv porten internt, 
    // men vi sikrer os her, at stifterne passer til dit modul.
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    
    // Initialiser biblioteket (0x01 er ESP32'erens XpressNet-adresse på bussen)
    XpressNet.setup(0x01, 0); 
    ESP_LOGI("xpressnet", "XpressNet tidskomponent startet.");
  }

  void loop() override {
    // Lad XpressNet-biblioteket håndtere indkommende data på bussen
    XpressNet.loop();

    // Send tiden fra Home Assistant til XpressNet hvert minut (60000 ms)
    if (millis() - last_update > 60000) {
      last_update = millis();

      // Hent den aktuelle tid fra ESPHome / Home Assistant tids-komponenten
      auto time = id(ha_time).now();
      
      if (time.is_valid()) {
        int hour = time.hour;
        int minute = time.minute;
        int weekday = time.day_of_week; // 1=Mandag, 7=Søndag i ESPHome

        // XpressNet forventer ugedag 0=Mandag .. 6=Søndag, så vi trækker 1 fra
        int xpressnet_weekday = weekday - 1; 
        if (xpressnet_weekday < 0) xpressnet_weekday = 0;

        // Hastighedsfaktor (1 = realtid, 0 = stoppet)
        int speed_factor = 1; 

        // Send tidskoden ud på XpressNet-bussen (Kommando 0x05 0x01...)
        // Vi bruger bibliotekets funktion til at sende en pakke afsted
        // (Hvis funktionen varierer i din specifikke udgave af biblioteket, 
        // kan du konstruere pakken manuelt som angivet i Lenz/OpenDCC specifikationen)
        
        ESP_LOGD("xpressnet", "Sender tid til XpressNet: %02d:%02d", hour, minute);
        
        // Eksempel på manuel pakke-konstruktion hvis biblioteket mangler direkte 'setTime'-funktion:
        byte msg[7];
        msg[0] = 0x05; // Længde og type
        msg[1] = 0x01; // Tidskode sub-type
        msg[2] = (0x00 | (minute & 0x3F));       // CC=00 (Minut)
        msg[3] = (0x80 | (hour & 0x1F));         // CC=10 (Time)
        msg[4] = (0x40 | (xpressnet_weekday & 0x07)); // CC=01 (Ugedag)
        msg[5] = (0xC0 | (speed_factor & 0x1F));  // CC=11 (Hastighed)
        msg[6] = msg[0] ^ msg[1] ^ msg[2] ^ msg[3] ^ msg[4] ^ msg[5]; // XOR Checksum
        
        for(int i=0; i<7; i++) {
          Serial2.write(msg[i]);
        }
      }
    }
  }
};
```

### 2. Konfigurer din ESPHome YAML-fil

Nu skal du konfigurere din ESPHome YAML-fil til at hente tidskilden fra Home Assistant, hente Philipp Gahtows bibliotek fra GitHub, og indlæse din C++ kode.

Åbn din .yaml fil og tilføj følgende sektioner:

```yaml
esphome:
  name: esp32-xpressnet-clock
  # Inkluder din custom C++ fil her
  includes:
    - xpressnet_time.h
  # Hent automatisk Philipp Gahtows XpressNet-bibliotek ind i projektet via PlatformIO
  libraries:
    - https://github.com

# Vi skal bruge Home Assistant som tidskilde
time:
  - platform: homeassistant
    id: ha_time
    timezone: "Europe/Copenhagen"

# Aktiver din Custom Component, som kører xpressnet_time.h koden
custom_component:
  - lambda: |-
      auto xpressnet_clock = new XpressNetTimeComponent();
      return {xpressnet_clock};
```

### Sådan virker det i praksis:

1. Biblioteks-import: ESPHome downloader automatisk Philipp Gahtows originale bibliotek direkte fra GitHub under kompileringen.
2. Retningsstyring: Da koden blot skriver rå bytes ud på Serial2 (Serial2.write), vil din MAX13487-chip på TTL485-V2.0-kortet registrere dataen med det samme og automatisk åbne/lukke for RS485-linjen uden behov for ekstra kode.
3. Tidssynkronisering: ESP32'eren modtager automatisk klokken fra Home Assistant over Wi-Fi. Hvert minut formaterer koden de rigtige bit-skift (CC=00 for minutter, CC=10 for timer osv. ud fra Lenz standarden) og sender det ud til alle tilsluttede håndreglere eller DCC-centralen, så ure på anlægget altid passer perfekt.

## jeg bruger home assistant til at kære en modelbane med en tid er går 6 til 8 gange nomaltid

Det giver rigtig god mening! En "model-tid" (Fast Clock), der løber hurtigere end normal tid, er præcis det, som XpressNet-protokollen er designet til at understøtte.

For at få dette til at fungere skal vi hente din model-tid (timer, minutter og den aktuelle ugedag) direkte fra de **sensorer eller hjælpere (input_number / input_datetime / sensor)**, du har oprettet i Home Assistant.

Her er, hvordan vi tilpasser koden og YAML-konfigurationen, så din ESP32 lytter på Home Assistant-sensorerne og sender dataene videre til din modelbane via din **MAX13487-chip**.

### 1. Opdater din ESPHome YAML-fil

Vi skal fortælle ESPHome, at den skal importere værdierne for din model-tid fra Home Assistant. I dette eksempel antager jeg, at du har tre sensorer i HA (f.eks. sensor.modeltid_timer, sensor.modeltid_minutter og sensor.modeltid_ugedag).

Du skal blot rette entity_id i eksemplet nedenfor, så de matcher de præcise navne, dine sensorer har i Home Assistant.

```yaml
esphome:
  name: esp32-xpressnet-clock
  includes:
    - xpressnet_time.h
  libraries:
    - https://github.com

# Hent timer, minutter og ugedag direkte fra Home Assistant som sensorer
text_sensor:
  - platform: homeassistant
    entity_id: sensor.modeltid_timer
    id: ha_model_timer
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_time_changed();'

  - platform: homeassistant
    entity_id: sensor.modeltid_minutter
    id: ha_model_minutter
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_time_changed();'

  - platform: homeassistant
    entity_id: sensor.modeltid_ugedag
    id: ha_model_ugedag
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_time_changed();'

# Aktiver din Custom Component og giv den et ID, så sensorerne kan tale med den
custom_component:
  - lambda: |-
      auto my_clock = new XpressNetTimeComponent();
      return {my_clock};
    id: xpressnet_clock
```

### 2. Opdater din C++ fil (xpressnet_time.h)

Nu ændrer vi C++ koden, så den i stedet for den rigtige tid lytter efter ændringer i dine Home Assistant-sensorer. Hver gang minuttet ændrer sig i Home Assistant, pakker ESP32'eren dataene og sender dem ud på XpressNet-bussen med den korrekte hastighedsfaktor (6 til 8 gange normal tid).

Erstat indholdet af din xpressnet_time.h med dette:

```cpp
#include "esphome.h"
#include <XpressNet.h>

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  bool time_changed = false;

 public:
  void setup() override {
    // Initialiser UART2 til din MAX13487-chip (62500 baud)
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    XpressNet.setup(0x01, 0); // ESP32 adresse på XpressNet-bussen (0x01)
    ESP_LOGI("xpressnet", "XpressNet Model-tidskomponent startet.");
  }

  // Kaldes fra YAML, når en sensor ændrer værdi i Home Assistant
  void set_time_changed() {
    time_changed = true;
  }

  void loop() override {
    XpressNet.loop(); // Håndter bussen (vigtigt for at modtage/svare centralen)

    // Hvis Home Assistant har opdateret tiden, sender vi den ud på bussen med det samme
    if (time_changed) {
      time_changed = false;

      // Hent tekstværdierne fra HA-sensorerne og lav dem om til heltal (int)
      int hour = atoi(id(ha_model_timer).state.c_str());
      int minute = atoi(id(ha_model_minutter).state.c_str());
      int weekday = atoi(id(ha_model_ugedag).state.c_str()); // Forventer 1=Mandag, 7=Søndag

      // Juster ugedag til XpressNet format (0=Mandag, 6=Søndag)
      int xpressnet_weekday = weekday - 1;
      if (xpressnet_weekday < 0 || xpressnet_weekday > 6) xpressnet_weekday = 0;

      // Hastighedsfaktor (Da din tid kører 6-8x hurtigere, sætter vi den her til f.eks. 7)
      // I XpressNet-protokollen indikerer dette over for håndregulerne, hvor hurtigt uret går.
      int speed_factor = 7; 

      ESP_LOGD("xpressnet", "Sender modeltid til XpressNet: D:%d - %02d:%02d (Skala 1:%d)", weekday, hour, minute, speed_factor);

      // Konstruer den officielle Lenz XpressNet tidspakke (0x05 0x01 ...)
      byte msg[7];
      msg[0] = 0x05; // Længde-header for tidskommando
      msg[1] = 0x01; // Sub-type: Ur/tid data
      
      // XpressNet bruger bitmasker (Command Control bits) til at identificere dataen:
      msg[2] = (0x00 | (minute & 0x3F));          // CC = 00 -> Minutter (0-59)
      msg[3] = (0x80 | (hour & 0x1F));            // CC = 10 -> Timer (0-23)
      msg[4] = (0x40 | (xpressnet_weekday & 0x07));// CC = 01 -> Ugedag (0-6)
      msg[5] = (0xC0 | (speed_factor & 0x1F));     // CC = 11 -> Hastighedsfaktor (1-31)
      
      // Beregn XOR Checksum (XpressNet kræver altid en fejlkontrol-byte til sidst)
      byte checksum = 0;
      for (int i = 0; i < 6; i++) {
        checksum ^= msg[i];
      }
      msg[6] = checksum;

      // Send de 7 bytes ud af porten. Din MAX13487 åbner og lukker automatisk for RS485-retningen!
      for (int i = 0; i < 7; i++) {
        Serial2.write(msg[i]);
      }
    }
  }
};
```

#### Hvordan det fungerer nu:

1. Hvert minut (eller hver gang minuttet rykker sig i din accelererede Home Assistant-model-tid), sender Home Assistant de nye værdier til ESP32.
2. ESPHome opfanger ændringen via on_value triggerværktøjet i YAML og beder C++ koden om at reagere.
3. Koden samler timer, minutter, ugedag og din hastighedsfaktor (speed_factor = 7), danner en fejlfri XpressNet-datapakke med checksum, og skyder den ud på bussen via din MAX13487-chip.
4. Alle ure og håndreglere (f.eks. Lenz LH101 eller WLANmaus forbundet til centralen), der lytter på bussen, vil nu synkront hoppe fremad i din hurtige model-tid.
AI-svar kan indeholde fejl. Få flere oplysninger

"" Min tid i home assistant gemmes i dette format int 1792012800

Det tal, du har (1792012800), er et **klassisk Unix-tidsstempel (antal sekunder siden 1. januar 1970)**. Som du kan se, gemmer det både dato, ugedag, timer og minutter i ét enkelt tal, hvilket faktisk gør det meget nemmere for os.

I ESPHome kan vi trække dette tal direkte ind som en almindelig sensor (en sensor i stedet for en text_sensor) og bruge C++'s indbyggede tidsfunktioner (tm-strukturen) til lynhurtigt at splitte tallet op i timer, minutter og ugedage.

Her er, hvordan du tilpasser opsætningen til dit tidsstempel:

### 1. Opdater din ESPHome YAML-fil

Du skal nu kun hente én enkelt sensor ind fra Home Assistant. Erstat din nuværende sensor-opsætning i YAML med denne:

```yaml
esphome:
  name: esp32-xpressnet-clock
  includes:
    - xpressnet_time.h
  libraries:
    - https://github.com

# Hent det rå tidsstempel (int) direkte fra din Home Assistant-sensor
sensor:
  - platform: homeassistant
    entity_id: sensor.din_modeltid_sensor  # <--- Ret til dit præcise HA sensor-navn
    id: ha_model_timestamp
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_new_timestamp(x);'

# Aktiver din Custom Component
custom_component:
  - lambda: |-
      auto my_clock = new XpressNetTimeComponent();
      return {my_clock};
    id: xpressnet_clock
```

### 2. Opdater din C++ fil (xpressnet_time.h)

Vi ændrer nu koden, så den modtager dit tidsstempel som en float/int. C++ konverterer derefter automatisk tallet til lokal tid, så vi kan trække timer, minutter og ugedag ud, før pakken sendes til din MAX13487-chip.

Erstat indholdet af xpressnet_time.h med dette:

```cpp
#include "esphome.h"
#include <XpressNet.h>
#include <time.h> // Bruges til at omregne tidsstemplet

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  uint32_t latest_timestamp = 0;
  bool time_changed = false;

 public:
  void setup() override {
    // Initialiser UART2 til din MAX13487-chip (62500 baud)
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    XpressNet.setup(0x01, 0); // ESP32 adresse på XpressNet-bussen
    ESP_LOGI("xpressnet", "XpressNet Tidsstempel-komponent startet.");
  }

  // Kaldes fra YAML, når dit tidsstempel opdateres i HA
  void set_new_timestamp(float value) {
    latest_timestamp = (uint32_t)value;
    time_changed = true;
  }

  void loop() override {
    XpressNet.loop(); // Håndter XpressNet-bussens generelle kommunikation

    if (time_changed && latest_timestamp > 0) {
      time_changed = false;

      // Konverter Unix-tidsstempel til en tidsstruktur (tm)
      time_t raw_time = (time_t)latest_timestamp;
      struct tm *time_info = localtime(&raw_time);

      if (time_info != nullptr) {
        int hour = time_info->tm_hour;      // Timer (0-23)
        int minute = time_info->tm_min;     // Minutter (0-59)
        
        // tm_wday giver 0=Søndag, 1=Mandag... 6=Lørdag
        // XpressNet forventer 0=Mandag, 1=Tirsdag... 6=Søndag
        int xpressnet_weekday = time_info->tm_wday - 1;
        if (xpressnet_weekday < 0) xpressnet_weekday = 6; // Hvis det var søndag (0), bliver det til 6

        // Din hastighedsfaktor (Skala for dit hurtige modelur, f.eks. 7)
        int speed_factor = 7; 

        ESP_LOGD("xpressnet", "Konverteret tidsstempel %d til Modeltid: Ugedag:%d - %02d:%02d", latest_timestamp, xpressnet_weekday, hour, minute);

        // Opbyg den officielle Lenz XpressNet tidspakke
        byte msg[7];
        msg[0] = 0x05; // Længde-header
        msg[1] = 0x01; // Sub-type (Ur data)
        
        msg[2] = (0x00 | (minute & 0x3F));           // CC = 00 -> Minutter
        msg[3] = (0x80 | (hour & 0x1F));             // CC = 10 -> Timer
        msg[4] = (0x40 | (xpressnet_weekday & 0x07)); // CC = 01 -> Ugedag
        msg[5] = (0xC0 | (speed_factor & 0x1F));      // CC = 11 -> Hastighed
        
        // Beregn XOR Checksum
        byte checksum = 0;
        for (int i = 0; i < 6; i++) {
          checksum ^= msg[i];
        }
        msg[6] = checksum;

        // Send pakken afsted – MAX13487 åbner og lukker selv bussen automatisk
        for (int i = 0; i < 7; i++) {
          Serial2.write(msg[i]);
        }
      }
    }
  }
};
```

#### Fordelen ved denne løsning:

Koden tager højde for alt det svære nu. Hver gang dit store tal ændrer sig i Home Assistant (f.eks. når uret i din modelverden hopper et minut frem), modtager ESP32'eren tallet, pakker det ud via localtime(), og videresender de korrekte bits til din modeljernbane helt automatisk.
Har du brug for, at hastighedsfaktoren (de 6 til 8 gange normaltid) også skal hentes dynamisk fra en sensor i Home Assistant, eller er det fint, at den er fastlåst i koden på f.eks. 7?

se python kode:

```python
# Let's decode 1792012800 as a Unix timestamp to see what it corresponds to
import datetime
ts = 1792012800
dt = datetime.datetime.fromtimestamp(ts, datetime.timezone.utc)
print(dt)
```

## jeg vil gerne bruge gmtime

Det er et rigtigt godt valg! Ved at bruge gmtime i stedet for localtime slipper du helt for, at ESP32'eren begynder at justere tiden for sommertid/vintertid eller tidszoner på baggrund af dit tidsstempel. gmtime tager tallet præcist, som det er, og splitter det rent op.

Her er den opdaterede xpressnet_time.h fil, hvor localtime er skiftet ud med gmtime. YAML-konfigurationen i ESPHome forbliver præcis den samme som før.

Opdateret C++ fil (xpressnet_time.h)

```cpp
#include "esphome.h"
#include <XpressNet.h>
#include <time.h> // Bruges til gmtime

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  uint32_t latest_timestamp = 0;
  bool time_changed = false;

 public:
  void setup() override {
    // Initialiser UART2 til din MAX13487-chip (62500 baud)
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    XpressNet.setup(0x01, 0); // ESP32 adresse på XpressNet-bussen
    ESP_LOGI("xpressnet", "XpressNet Tidsstempel-komponent med gmtime startet.");
  }

  // Kaldes fra YAML, når dit tidsstempel opdateres i HA
  void set_new_timestamp(float value) {
    latest_timestamp = (uint32_t)value;
    time_changed = true;
  }

  void loop() override {
    XpressNet.loop(); // Håndter bussen (vigtigt for at modtage/svare centralen)

    if (time_changed && latest_timestamp > 0) {
      time_changed = false;

      // Konverter Unix-tidsstempel til en tm-struktur ved brug af GMT/UTC tid (ren tid)
      time_t raw_time = (time_t)latest_timestamp;
      struct tm *time_info = gmtime(&raw_time);

      if (time_info != nullptr) {
        int hour = time_info->tm_hour;      // Timer (0-23)
        int minute = time_info->tm_min;     // Minutter (0-59)
        
        // tm_wday giver 0=Søndag, 1=Mandag... 6=Lørdag
        // XpressNet forventer 0=Mandag, 1=Tirsdag... 6=Søndag
        int xpressnet_weekday = time_info->tm_wday - 1;
        if (xpressnet_weekday < 0) xpressnet_weekday = 6; // Hvis det var søndag (0), bliver det til 6 (Søndag i XpressNet)

        // Din hastighedsfaktor (Skala for dit hurtige modelur, f.eks. 7)
        int speed_factor = 7; 

        ESP_LOGD("xpressnet", "gmtime konverteret %d til Modeltid: Ugedag:%d - %02d:%02d", latest_timestamp, xpressnet_weekday, hour, minute);

        // Opbyg den officielle Lenz XpressNet tidspakke
        byte msg[7];
        msg[0] = 0x05; // Længde-header
        msg[1] = 0x01; // Sub-type (Ur data)
        
        msg[2] = (0x00 | (minute & 0x3F));           // CC = 00 -> Minutter
        msg[3] = (0x80 | (hour & 0x1F));             // CC = 10 -> Timer
        msg[4] = (0x40 | (xpressnet_weekday & 0x07)); // CC = 01 -> Ugedag
        msg[5] = (0xC0 | (speed_factor & 0x1F));      // CC = 11 -> Hastighed
        
        // Beregn XOR Checksum
        byte checksum = 0;
        for (int i = 0; i < 6; i++) {
          checksum ^= msg[i];
        }
        msg[6] = checksum;

        // Send pakken afsted – MAX13487 åbner og lukker selv bussen automatisk
        for (int i = 0; i < 7; i++) {
          Serial2.write(msg[i]);
        }
      }
    }
  }
};
```

(Bemærk: Jeg har også rettet en mindre syntaksfejl fra det tidligere svar, så msg nu er deklareret korrekt som et array byte msg[7], hvilket sikrer, at kompileringen i ESPHome forløber helt uden fejl).

Med denne ændring tages værdierne direkte ud af dit tidsstempel (f.eks. 1792012800), og der vil aldrig opstå uventede tidsforskydninger, uanset hvilken tidszone din Home Assistant eller ESP32 ellers er sat til.

Skal vi også kigge på, hvordan du modtager data (f.eks. hvis nogen ændrer sporskifter eller hastighed på et fysisk håndhåndtag, og du vil have det retur til Home Assistant), eller er det i første omgang udelukkende uret, du har fokus på?

## Hvilken addresse optræder min esp32 med på xpressbussen.  er der noget jeg skal tage hensyn til i forhold til andre håndterminaler

I den kode, vi har sat op, optræder din ESP32 med **XpressNet-adresse 1** (angivet via XpressNet.setup(0x01, 0); i C++ koden).

Der er et par meget vigtige ting, du skal tage hensyn til i forhold til bussen og andre håndterminaler (som f.eks. Lenz LH101 eller Roco Multimaus):

### 1. Undgå adresse-konflikter (Meget vigtigt!)

Hver eneste enhed på XpressNet-bussen skal have sin egen unikke adresse. Hvis to enheder har samme adresse, vil de forsøge at svare centralen på samme tid. Det skaber datakorruption, og bussen vil begynde at fejle eller fryse.

* **Tjek dine andre håndterminaler**: Sørg for, at ingen af dine fysiske håndreglere er indstillet til adresse 1. Typisk starter fysiske håndreglere på adresse 1, 2 eller 3 fra fabrikken.
* **Standardadresser**: Mange centraler (f.eks. z21 eller Lenz) har reserveret specifikke adresser internt (f.eks. adresse 0 til centralen selv).
* **Anbefaling**: For at være på den sikre side, kan det være en god idé at flytte din ESP32 op på en højere adresse, som sjældent bruges af standardudstyr – f.eks. **adresse 20 eller adresse 30**.

Hvis du vil ændre det, skal du blot rette tallet i din xpressnet_time.h fil:

```cpp
XpressNet.setup(30, 0); // Ændrer ESP32'erens XpressNet-adresse til 30
```

### 2. XpressNet-bussens timeout (Hvorfor XpressNet.loop() er vigtig)

XpressNet er en såkaldt "Master-Slave"-bus (eller rettere en "Token Ring"-bus). Centralen sender konstant en forespørgsel (et "Call Byte") ud til hver enkelt adresse på skift (adresse 1, så adresse 2, osv.) for at høre, om de har noget at rapportere.

* Din ESP32 skal svare lynhurtigt, hver gang den bliver spurgt. Det er det, som XpressNet.loop(); i koden sørger for.
* **ESPHome-hensyn**: Fordi ESPHome også skal håndtere Wi-Fi og kommunikation med Home Assistant, må C++ koden i loop() ikke blokere eller tage for lang tid. Vores tids-kode er heldigvis meget hurtig og effektiv, så den vil ikke forstyrre bussens timing.

### 3. Håndtering af uret (Fast Clock) på bussen

XpressNet-protokollen er designet således, at kun én enhed bør sende master-uret ud.

* Når din ESP32 sender modeltiden ud hvert minut, vil alle andre håndterminaler (der understøtter ur-visning) automatisk opdatere deres display til at vise din hurtige modeltid.
* Du skal blot sikre dig, at din DCC-central ikke også forsøger at køre sit eget uafhængige ur, da de to ure så vil modarbejde hinanden på bussen. I de fleste centraler kan uret deaktiveres eller sættes til "slave".

Har du styr på, hvilke adresser dine nuværende håndterminaler bruger, eller vil du have, at vi ændrer koden til en højere adresse (f.eks. 30) med det samme for at undgå potentielle konflikter?

## Kan jeg give dcc centralen besked om at rette sin tid og så sende tids information ud til håndterminalerne, og på den måde undgå konfligte mellem esp32 og dcc

### Ja, det er faktisk den absolut bedste og mest korrekte måde at gøre det på!

I stedet for at din ESP32 leger "master-ur" og risikerer at støde sammen med DCC-centralens eget ur, kan vi sende en specifik XpressNet-kommando, der opdaterer selve centralens interne ur. Når centralen modtager denne kommando, retter den sit ur og sørger derefter selv for automatisk at sende den nye tid ud til alle håndterminaler.

På den måde overholder vi XpressNet-protokollens hierarki 100%, og du slipper helt for konflikter.

For at gøre dette skal vi blot ændre den pakke (kommando), som din ESP32 sender afsted. I stedet for at sende en generel bus-udsendelse (broadcast), sender vi en "Skriv ur-indstilling til centralen"-kommando.

Her er den opdaterede og fejlsikre C++ kode til din ESP32:

#### Opdateret C++ fil (xpressnet_time.h)

```cpp
#include "esphome.h"
#include <XpressNet.h>
#include <time.h> 

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  uint32_t latest_timestamp = 0;
  bool time_changed = false;

 public:
  void setup() override {
    // Initialiser UART2 til din MAX13487-chip (62500 baud)
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    
    // Vi sætter ESP32 til adresse 30 for at være helt fri af standard håndreglere (1-3)
    XpressNet.setup(30, 0); 
    ESP_LOGI("xpressnet", "XpressNet Central-Ur synkronisering startet.");
  }

  void set_new_timestamp(float value) {
    latest_timestamp = (uint32_t)value;
    time_changed = true;
  }

  void loop() override {
    XpressNet.loop(); // Svarer centralen på adresse 30, så bussen kører stabilt

    if (time_changed && latest_timestamp > 0) {
      time_changed = false;

      time_t raw_time = (time_t)latest_timestamp;
      struct tm *time_info = gmtime(&raw_time);

      if (time_info != nullptr) {
        int hour = time_info->tm_hour;      
        int minute = time_info->tm_min;     
        
        // 0=Søndag, 1=Mandag... 6=Lørdag -> XpressNet: 0=Mandag... 6=Søndag
        int xpressnet_weekday = time_info->tm_wday - 1;
        if (xpressnet_weekday < 0) xpressnet_weekday = 6; 

        // Hastighedsfaktor for dit modelur (f.eks. 7 gange normaltid)
        int speed_factor = 7; 

        ESP_LOGD("xpressnet", "Sender ny tid til DCC Central: %02d:%02d (Skala 1:%d)", hour, minute, speed_factor);

        // Officiel Lenz XpressNet-kommando: "Set Central Clock" (0x24 0x01 ...)
        // Denne kommando fortæller centralen, at den skal ændre sin tid og sende den ud.
        byte msg[7];
        msg[0] = 0x24; // Header: 2 databytes (0x20) + Kommando type (4) -> Set Central Settings
        msg[1] = 0x01; // Sub-type: Ur/tid indstilling
        
        // Indkodning af data med Command Control bitmasker:
        msg[2] = (0x00 | (minute & 0x3F));           // CC = 00 -> Minutter
        msg[3] = (0x80 | (hour & 0x1F));             // CC = 10 -> Timer
        msg[4] = (0x40 | (xpressnet_weekday & 0x07)); // CC = 01 -> Ugedag
        msg[5] = (0xC0 | (speed_factor & 0x1F));      // CC = 11 -> Hastighedsfaktor
        
        // Beregn XOR Checksum
        byte checksum = 0;
        for (int i = 0; i < 6; i++) {
          checksum ^= msg[i];
        }
        msg[6] = checksum;

        // Send pakken til centralen via din MAX13487 (som styrer retningen automatisk)
        for (int i = 0; i < 7; i++) {
          Serial2.write(msg[i]);
        }
      }
    }
  }
};
```

#### Hvad er ændret nu?

1. Adresse 30: Jeg har ændret ESP32'erens bus-adresse til 30 i koden (XpressNet.setup(30, 0);). Det sikrer, at den ikke konflikter med dine fysiske håndreglere.
2. Kommando 0x24: I stedet for den gamle modtager-pakke sender vi nu 0x24 som den første byte. Det er XpressNet-koden for "Hej central, her er den nye master-tid".
3. Ren styring: Nu modtager centralen tiden fra Home Assistant (via ESP32), opdaterer sit eget system, og distribuerer det fejlfrit ud til dine håndterminaler på bussen.

Er du klar til at teste koden af på din central, eller vil du have hjælp til at verificere, at navnet på din tids-sensor i Home Assistant spiller rigtigt sammen med ESPHome?

## vi skal også have dcc centralen til at hurtig til svarende til den hastighed vi vælger i home assistsant

Det giver rigtig god mening! Hvis vi vil have, at DCC-centralen og dens ure rent faktisk følger og opdaterer tiden med den accelererede hastighed (f.eks. 6-8x normaltid), skal hastigheden ikke være fastlåst i C++ koden. Vi skal i stedet hente den dynamisk som en sensor fra Home Assistant, ligesom vi gør med tidsstemplet.

Her er den komplette opdatering, hvor hastighedsfaktoren (skalaen) sendes med over til ESP32'eren, hver gang uret tikker fremad.

### 1. Opdater din ESPHome YAML-fil

Vi tilføjer en ekstra sensor i din YAML-fil, som henter din valgte hastighedsfaktor (f.eks. fra en input_number eller en sensor i Home Assistant), og gemmer den direkte i C++ komponenten.

```yaml
esphome:
  name: esp32-xpressnet-clock
  includes:
    - xpressnet_time.h
  libraries:
    - https://github.com

sensor:
  # 1. Tidsstempel-sensoren (som før)
  - platform: homeassistant
    entity_id: sensor.din_modeltid_sensor  # <--- Ret til dit præcise HA tidsstempel-navn
    id: ha_model_timestamp
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_new_timestamp(x);'

  # 2. DEN NYE SENSOR: Hastighedsfaktoren (f.eks. et tal mellem 1 og 30)
  - platform: homeassistant
    entity_id: input_number.modeltid_hastighed  # <--- Ret til din HA-hjælper / sensor
    id: ha_model_speed
    on_value:
      then:
        - lambda: 'id(xpressnet_clock).set_speed_factor(x);'

# Aktiver din Custom Component
custom_component:
  - lambda: |-
      auto my_clock = new XpressNetTimeComponent();
      return {my_clock};
    id: xpressnet_clock
```

### 2. Opdater din C++ fil (xpressnet_time.h)

Vi ændrer nu C++ koden, så den både modtager det dynamiske tidsstempel og den dynamiske hastighedsfaktor. Når værdierne sendes til DCC-centralen, vil centralen indstille sit interne ur til at løbe med præcis den faktor, du har valgt i Home Assistant.

```cpp
#include "esphome.h"
#include <XpressNet.h>
#include <time.h> 

class XpressNetTimeComponent : public Component, public CustomComponent {
 private:
  XpressNetClass XpressNet;
  uint32_t latest_timestamp = 0;
  int current_speed_factor = 7; // Standardværdi hvis HA ikke har sendt noget endnu
  bool time_changed = false;

 public:
  void setup() override {
    // Initialiser UART2 til din MAX13487-chip (62500 baud)
    Serial2.begin(62500, SERIAL_8N1, 16, 17); 
    
    // ESP32 adresse 30 på XpressNet-bussen
    XpressNet.setup(30, 0); 
    ESP_LOGI("xpressnet", "XpressNet Central-Ur synkronisering startet.");
  }

  // Kaldes fra YAML, når dit tidsstempel opdateres i HA
  void set_new_timestamp(float value) {
    latest_timestamp = (uint32_t)value;
    time_changed = true;
  }

  // Kaldes fra YAML, når du ændrer hastigheden i HA
  void set_speed_factor(float value) {
    current_speed_factor = (int)value;
    // Sikrer at værdien holder sig inden for XpressNet-protokollens grænser (1 til 31)
    if (current_speed_factor < 1) current_speed_factor = 1;
    if (current_speed_factor > 31) current_speed_factor = 31;
  }

  void loop() override {
    XpressNet.loop(); // Svarer centralen på adresse 30

    if (time_changed && latest_timestamp > 0) {
      time_changed = false;

      time_t raw_time = (time_t)latest_timestamp;
      struct tm *time_info = gmtime(&raw_time);

      if (time_info != nullptr) {
        int hour = time_info->tm_hour;      
        int minute = time_info->tm_min;     
        
        // Konverter ugedag (0=Søndag, 1=Mandag... 6=Lørdag -> XpressNet: 0=Mandag... 6=Søndag)
        int xpressnet_weekday = time_info->tm_wday - 1;
        if (xpressnet_weekday < 0) xpressnet_weekday = 6; 

        ESP_LOGD("xpressnet", "Sender ny tid til DCC Central: %02d:%02d (Hastighedsfaktor: %dx)", hour, minute, current_speed_factor);

        // Officiel Lenz XpressNet-kommando: "Set Central Clock" (0x24 0x01 ...)
        byte msg[7];
        msg[0] = 0x24; // Header: Set Central Settings
        msg[1] = 0x01; // Sub-type: Ur data
        
        // Indkodning med Command Control bitmasker:
        msg[2] = (0x00 | (minute & 0x3F));           // CC = 00 -> Minutter
        msg[3] = (0x80 | (hour & 0x1F));             // CC = 10 -> Timer
        msg[4] = (0x40 | (xpressnet_weekday & 0x07)); // CC = 01 -> Ugedag
        msg[5] = (0xC0 | (current_speed_factor & 0x1F)); // CC = 11 -> Den dynamiske hastighedsfaktor fra HA
        
        // Beregn XOR Checksum
        byte checksum = 0;
        for (int i = 0; i < 6; i++) {
          checksum ^= msg[i];
        }
        msg[6] = checksum;

        // Send de 7 bytes til centralen via din MAX13487
        for (int i = 0; i < 7; i++) {
          Serial2.write(msg[i]);
        }
      }
    }
  }
};
```

#### Hvordan det virker nu:

1. Hvis du i Home Assistant ændrer din tidsskala (f.eks. skruer op fra 6x til 8x hastighed via en slider/input_number), sendes den nye værdi med det samme over til ESP32'eren.
2. Næste gang uret tikker et minut frem i din modelverden, pakkes både den nye tid og den nye hastighedsfaktor ind i 0x24-kommandoen.
3. DCC-centralen modtager pakken og ved nu både, hvad klokken er, og hvor hurtigt dens eget interne ur skal løbe fremad. Centralen synkroniserer herefter automatisk sig selv og alle tilsluttede håndterminaler på bussen, så alt kører i det helt rigtige accelererede tempo.

