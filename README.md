# Modeltog-Ursystem

How to make a Time system for Model trains

## Hvad er det jeg vil opnå med dette ur system

* Jeg vil gerne lave et ursystem som vi kan bruge når vi har køreplands dage, på køreplansdage starter vores urer kl: 03:55 og bevæger sig med model tid som er 6-8 gange hurtigere end nomaltid, når køreplanen når kl: 12:00 holder vi en kaffepause, så her hav vi behov for at kunne pause tiden, og starte den igen så fortsætter køreplanen til vi får det sidste tog til ets endestation omkring køreplanstid Kl: 23:00.
* Opstår der problemer med rangerer tog op og vi kommer mere 10min efter køreplanen skal vi stoppe tiden, så alle kan følge med, den der stopper tiden er også den som skal starte den igen.
* vi har en blanding af gamle central styret urer, og nye computer styret urer, så når vi er færdige med en køreplandsdag, skal vi have de gamle urer kørt frem til start tiden 03:55, så de er klar til næste køredag.
Da systemet ved hvor mange minuter der er til næste dag Kl: 03:55, sender den så det rigtige antal minut signaler ud til de gamle system urer, ved betjening fra master kontrolleren.
* jeg vil anbringe små ur kontroller ved hver station med et lille elektronisk ur som viser tiden analog, og denne til skalvære den samme som på de gamle urer, der skal også være en pause kanp på uret, trykker man på pause knappen skal alle display vise hvem der stoppede tiden, og vedkommende er også den som kan starte tiden igen.
* Jeg bad AI om hjælp til denne opgave og herunder kan i se dialogen med Google's AI.

## Hvad bruger jeg til at styre det hele ?

* Da vi allerede bruger Home Assistant og ESPHome til vores sikrings system, er det oplagt at brug den samme Home Assistant til dette, og lave nogle ESP32 med display og trykknapper til stationerne, og en anden ESP32 med med H-Bridge kontroller til at sende pulser til system urerne.

## [For nu klik denne link](https://github.com/sekt1953/Modeltog-Sikringsanlaeg/tree/main/Project/Clock)