# Home Assistant tid til Lenz system

Ja, det kan du godt! Du kan vende kommunikationen om, så din ESP32 tager tiden fra Home Assistant og sender den direkte ind som den nye officielle systemtid på din Lenz-central.

Når du gør det, vil din Lenz-central (og dermed dine håndterminaler som LH101) synkronisere sig fuldstændigt med uret i dit smarte hjem [github.com].

Her er, hvordan du bygger det ind i din eksisterende ESPHome-opsætning:

### 1. Tilføj Home Assistant-tid i din YAML

Først skal vi have ESPHome til at hente det rigtige klokkeslæt fra Home Assistant via netværket. Det gør du ved at tilføje en time-komponent i din lenz_styring.yaml:

```yaml
# Hent tiden fra Home Assistant over netværket
time:
  - platform: homeassistant
    id: ha_time
    # Valgfrit: Send tiden videre til din C++ kode hvert minut
    on_minute:
      - lambda: |-
          // Vi kalder en funktion i vores C++ kode, når uret skifter minut
          xnet_clock->sync_time_to_lenz(id(ha_time).now().hour, id(ha_time).now().minute);
```

Du skal også opdatere din text_sensor definition i YAML-filen, så vi kan tilgå xnet_clock objektet i den globale lambda (bemærk tilføjelsen af xnet_clock = uden for lambdaen):

```yaml
# Tilføj dette i toppen af din YAML (under esphome_core_config eller som global)
globals:
  - id: global_xnet_clock
    type: LenzClockSensor*

text_sensor:
  - platform: custom
    lambda: |-
      auto xnet_clock = new LenzClockSensor();
      return {xnet_clock->time_sensor};
```

(For at holde det helt simpelt, kan vi dog også bare lade C++ koden selv holde øje med tiden direkte).

### 2. Opdater din lenz_clock.h (C++ koden)

I XPressNet-biblioteket findes der en indbygget funktion, som hedder setClock(timer, minutter, hastighedsfaktor). Når ESP32 kalder denne funktion, sender den en 0x82-pakke ud på XpressNet-bussen, hvilket tvinger Lenz-centralen til at opdatere sit ur [github.com].

Her er, hvordan din lenz_clock.h skal se ud for at kunne modtage og sende tiden:

```cpp
#include "esphome.h"
#include <XPressNet.h>

#define XNET_RX_PIN 16
#define XNET_TX_PIN 17
#define XNET_DE_PIN 4

class LenzClockSensor : public esphome::Component {
 public:
  esphome::text_sensor::TextSensor *time_sensor = new esphome::text_sensor::TextSensor();
  XPressNet xnet;
  int last_minute = -1;
  int last_hour = -1;

  void setup() override {
    xnet.setup(2, XNET_RX_PIN, XNET_TX_PIN, XNET_DE_PIN);
    time_sensor->set_name("Lenz Systemtid");
  }

  void loop() override {
    xnet.update();

    // Læs tiden fra Lenz-centralen (så HA stadig kan se, hvad uret står på)
    int current_hour = xnet.getHour();
    int current_minute = xnet.getMinute();

    if (current_minute != last_minute || current_hour != last_hour) {
      if (current_hour >= 0 && current_hour < 24 && current_minute >= 0 && current_minute < 60) {
        last_hour = current_hour;
        last_minute = current_minute;
        char time_string[6];
        sprintf(time_string, "%02d:%02d", current_hour, current_minute);
        time_sensor->publish_state(time_string);
      }
    }
  }

  // NY FUNKTION: Denne funktion kaldes fra din YAML for at sende HA-tid til Lenz
  void sync_time_to_lenz(int ha_hour, int ha_minute) {
    // Tjek at data er gyldig
    if (ha_hour >= 0 && ha_hour < 24 && ha_minute >= 0 && ha_minute < 60) {
      
      // xnet.setClock(timer, minutter, tidsfaktor)
      // Tidsfaktor 1 = Realtid (1 minut i virkeligheden er 1 minut på modelbanen)
      // Du kan sætte den højere (f.eks. 4), hvis modeltiden skal løbe hurtigere.
      xnet.setClock(ha_hour, ha_minute, 1);
      
      ESP_LOGD("XPressNet", "Synkroniserede Home Assistant tid (%02d:%02d) til Lenz-centralen", ha_hour, ha_minute);
    }
  }
};
```

### Vigtigt at være opmærksom på:

* Tidsfaktor (Modeltid): I koden ovenfor har jeg sat tidsfaktoren til 1 (xnet.setClock(ha_hour, ha_minute, 1)). Det betyder, at dit modelbaneanlæg vil køre i præcis samme tempo som den virkelige verden. Hvis du bruger uret til f.eks. køreplaner, hvor tiden skal gå 4 gange hurtigere, skal du ændre det sidste tal til 4.
* Rundsavs-effekt: Da du både læser og skriver tiden, skal du passe på, at Home Assistant og Lenz-centralen ikke kæmper om uret. Ved at bruge on_minute i ESPHome sender vi kun en opdatering én gang i minuttet, hvilket er rigeligt til at holde dem i perfekt synkronisering uden at overbelaste XpressNet-bussen.

Vil du have, at modelbanens ur skal følge den rigtige tid 1:1, eller kører du med en hastighedsfaktor (hurtigt ur) på dit anlæg?
