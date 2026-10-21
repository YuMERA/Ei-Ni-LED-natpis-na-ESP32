# CLAUDE.md

Projekat je na srpskom — komentari, dokumentacija i komunikacija sa korisnikom.
Tekst za korisnika ide sa dijakritikom; komentari u firmware-u su bez nje.

## Cilj

Prepravka starog LED panela Ei Niš, model **INFOS 1802** (ranije pogrešno
pročitano kao „INF 1505/1508“) — izbaciti originalni MCU deo i panel pogoniti
sa ESP32. Namena: skrolujući natpis sa porukama koje se unose preko web panela
na ESP32, plus meteo podaci kao jedna od stavki.

Panel: jednoredni crveni dot-matrix u dugačkom kućištu.

O ovom modelu nema podataka na internetu. Korisnik ima originalnu
dokumentaciju: „Priručnik programiranja“ i „Uputstvo za instaliranje“.
Iz njih, kao i iz slika unutrašnjosti, utvrditi rezoluciju, interfejs i
protokol.

Alternativa koju treba odmeriti kad se vidi priručnik: ako stara elektronika
radi i panel prima poruke serijski (RS-232/RS-485), ESP32 može da ostane
spolja i da šalje poruke po originalnom protokolu, bez diranja ploče.

## Pravila

**Ne pokretati Arduino kompajliranje.** Korisnik ga radi sam. Piši izmene, ali
reci šta nisi mogao da proveriš.

**Korisnik radi sve `git push` operacije.** Commit je u redu, push nije.

**Server se ne menja odavde.** Meteo podaci dolaze iz Output API-ja u
`../mera-HUB-app` (`GET /api/output/:id`, token u `x-mera-output-token`).

## Pretpostavke o hardveru (proveriti na slikama)

Tipično za panele iz tog perioda:
- Kolone: lanac pomeračkih registara (CD4094, 74LS164/74HC595, UCN5821/5895),
  ulančanih po modulima.
- Redovi: multipleks preko dekodera (74LS138/74LS145) i tranzistora
  (BD140, TIP127, ULN2803), 1/7 ili 1/8.
- Logika: 8051/Z80/6502 + EPROM + SRAM — to se izbacuje.
- Interfejs MCU → LED moduli: DATA, CLK, LATCH, OE i A0–A2 (izbor reda).

Plan:
1. Naći konektor između MCU ploče i LED modula, identifikovati signale
   (po mogućstvu logičkim analizatorom dok stara elektronika radi).
2. Prilagođavanje 3,3 V → 5 V (74HCT245 / 74AHCT125).
3. Firmware: SPI (DMA) za kolone, tajmer za multipleks redova na jezgru 1,
   osvežavanje ≥ ~100 Hz.
4. Web panel: lista poruka (tekst, brzina, smer, trajanje) u LittleFS/
   Preferences; meteo ekran kao stavka liste.

## Iz `../mera-ledMatrix app` može da se preuzme

- `MeraData.h` — HTTP task na jezgru 0 koji čita Output API, displej uzima
  kopiju pod mutex-om.
- Ideja `Frame` bafera iz `Display.h`: crtež u memoriji, backend ga šalje na
  hardver. Ovde multipleks mora da radi u prekidu, nezavisno od crtanja.
- Font, ako panel ima 8 redova; za 16 redova treba veći.
