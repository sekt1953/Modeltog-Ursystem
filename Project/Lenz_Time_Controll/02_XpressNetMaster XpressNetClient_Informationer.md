# XpressNetMaster / XpressNetClient informationer

## hvilke andre informationer kan jeg hente med XpressNetMaster / XpressNetClient

Med XPressNet-biblioteket (ofte refereret til som XPressNetMaster / XPressNetClient) kan du hente stort set alle de informationer, som din Lenz-central sender ud på bussen. Da din ESP32 lytter som en "klient" (en ekstra håndterminal), kan den overvåge alt, hvad der sker på modelbanen.

Her er de mest relevante informationer, du kan hente ud til dit Home Assistant-system:

### 1. Sporstatus (Strøm og nødstop)

Du kan overvåge, om banen er tændt, eller om der er trykket på nødstop. Det er genialt til at lave en status-indikator i Home Assistant.

* Hvad du kan hente:
  * Er spændingen slået til (Track Power On)?
  * Er der trykket på nødstop (Emergency Stop / Estop)?
  * Er der en kortslutning på banen?
* Biblioteksfunktioner: xnet.getPowerMode() (retunerer statusser som Normal, Off eller Service Mode).

### 2. Sporskifter, signaler og tilbehør (Turnout / Accessory Status)

Hver gang et sporskifte eller et signal skifter position (enten via din håndterminal eller et computerprogram som f.eks. iTrain/JMRI), sendes dette ud på XpressNet.

* Hvad du kan hente: 
  * Den aktuelle fysiske position af sporskifter/signaler (f.eks. om sporskifte 12 er sat til "ligeud" eller "afvigende").
  * Biblioteksfunktioner: Ved at bruge en callback-funktion som notifyXNetTrnt() i din C++ kode, får du besked i det sekund, et sporskifte aktiveres. Du kan konvertere dette til binary_sensor eller switch komponenter i Home Assistant.

### 3. Lokomotiv-informationer (Loko-status)

Hvis du vil vide, hvad der kører på banen lige nu, kan din ESP32 lytte med på de lokomotiver, som håndterminalerne styrer.

* Hvad du kan hente:
  * Hastighedstrin (Speed steps - f.eks. om et tog kører på trin 45 ud af 128).
  * Kørselsretning (Fremad eller bagud).
  * Funktioner (F0 - F28): Om lyset (F0), lyden (F1) eller andre funktioner er tændt på et specifikt lokomotiv.
* Biblioteksfunktioner: xnet.getLocoSpeed() og xnet.getLocoFunc() (kræver at du forespørger på den specifikke DCC-adresse, eller lytter til ændringer via callbacks).

### 4. Centralspecifikke informationer

Du kan udlæse data om selve din Lenz-central (f.eks. LZV200 eller ældre LZV100).

* Hvad du kan hente:
  * Softwareversion på centralen (f.eks. version 3.6 eller 4.0).
  * Centralens ID (hvilken type hardware der køres på).
* Biblioteksfunktioner: xnet.getV200Version() / xnet.getCentralVersion().

### 5. Avanceret: Tilbagemeldinger (Feedback / S88 via XpressNet)

Hvis du har Lenz tilbagemeldingsmoduler (f.eks. Lenz LR101) koblet til din central for at registrere, hvor togene befinder sig på skinnerne (blokdetektering), sendes disse data også over XpressNet.

* Hvad du kan hente: Hvilke sporafsnit der er optaget eller ledige.
* Biblioteksfunktioner: xnet.getFeedback() (kan bruges til at lave sensorer i Home Assistant, der viser præcis, hvor togene er).

## Hvordan bringer du det ind i ESPHome?

Hvis du eksempelvis også ville have Sporstatus (On/Off) op i Home Assistant som en binary_sensor, ville du i din lenz_clock.h tilføje et tjek i bunden af din loop():

```cpp
// Eksempel på at hente Power-status i loop()
int power = xnet.getPowerMode(); 
// Hvis power == 1 er strømmmen tændt, hvis 0 er den slukket/nødstop
```

Du kan derved forvandle din ESP32 til en fuldblods "Lenz-til-Home Assistant" bro, der både håndterer uret, overvåger sporskifter og holder øje med, om strømmen kortslutter på modelbanen!

