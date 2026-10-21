# CLAUDE.md

Projekat je na srpskom — komentari, dokumentacija i komunikacija sa korisnikom.
Tekst za korisnika ide sa dijakritikom; komentari u firmware-u su bez nje.

## Cilj

Prepravka starog LED panela Ei Niš, model **INFOS 1802** (ranije pogrešno
pročitano kao „INF 1505/1508“) — izbaciti originalni MCU deo i panel pogoniti
sa ESP32. Namena: skrolujući natpis sa porukama koje se unose preko web panela
na ESP32, plus meteo podaci kao jedna od stavki.

Panel: jednoredni crveni dot-matrix u dugačkom kućištu.

O ovom modelu nema podataka na internetu. Uz panel je išao zaseban kontroler
sa tastaturom za unos teksta; korisnik ga nema. Postoje originalni
„Priručnik programiranja“ i „Uputstvo za instaliranje“, ali se odnose na rad
sa tim kontrolerom, pa nisu primarni izvor.

Odluka korisnika: zadržati horizontalne/vertikalne drajvere za LED, a logiku
zameniti sa ESP32 (unos poruka preko web panela/USB). Sve se utvrđuje iz
slika ploče.

Korisnik ima **dva panela**:
- **Panel A** — započeto rasklapanje: deo MCU komponenti je davno skinut.
  Ovaj se prepravlja.
- **Panel B** — netaknut i radi: po uključenju ispisuje ranije isprogramirane
  poruke (dakle u samom panelu postoji MCU i memorija). Služi kao referenca:
  na njemu se, dok radi, logičkim analizatorom snimaju DATA/CLK/LATCH/OE i
  izbor reda. Ne dirati ga dok se signali ne snime.

## Pravila

**Ne pokretati Arduino kompajliranje.** Korisnik ga radi sam. Piši izmene, ali
reci šta nisi mogao da proveriš.

**Korisnik radi sve `git push` operacije.** Commit je u redu, push nije.

**Server se ne menja odavde.** Meteo podaci dolaze iz Output API-ja u
`../mera-HUB-app` (`GET /api/output/:id`, token u `x-mera-output-token`).

## Hardver — utvrđeno sa slika (2026-10-21)

Slike su u `foto/` (HEIC sa Samsung telefona, ne ide u git). Konverzija u JPG
radi preko WPF dekodera u PowerShell-u (`BitmapDecoder` → `JpegBitmapEncoder`),
Python/ImageMagick nisu instalirani. Slike koje stignu u chat su smanjene na
400×180 — za čitanje oznaka uvek koristiti originale iz `foto/`.

**MCU ploča** (oznaka „5359155 A-U1“), sve CMOS, RCA COSMAC familija:
- CDP1802ACE — CPU (RCA 1802).
- NMC27C64Q — EPROM 8K, nalepnica „INF 1802/08 V.2.0“ (odatle ranija zabuna
  oko „1505/1508“).
- Prazno mesto „6264“ — SRAM 8K (poruke), skinuo ga je korisnik. 4 ležišta
  „1,2V“ za NiCd bateriju — baterija nikad nije ni bila.
- CDP1852CE — 8-bitni I/O port; pored njega prazno mesto „1852“ za još jedan,
  sa vodovima ka tranzistorima redova (skinuo korisnik).
- Slikana ploča je iz **panela A** (rasturenog). LED moduli na njemu nisu
  dirani.
- CDP1853CE — dekoder 1-od-8 (N linije 1802).
- CDP1854ACE — UART; MAX232 — RS-232 nivoi; HD-4702 (Harris) — generator
  baud-a, kristal 2,4576 MHz, kratkospojnici „1200“/„9600“. Tuda je
  stizao tekst sa spoljnog kontrolera sa tastaturom.
- TC4049BP (inverter/bafer), 2× HCF4042BE (D latch), HCF4013BE (D FF).
- LT7805CV — 5 V za logiku.
- Redovi: 8 TO-220 tranzistora T1–T8 (T1/3/5/7 na ploči, T2/4/6/8 na
  aluminijumskom nosaču uz ivicu). Oznaka tipa nije pročitana. Do redova na
  modulima idu žute žice.

**LED modul** (oznaka „5357996“, datum „29 89“):
- 8 redova (8 LED po visini) — font iz `mera-ledMatrix` odgovara.
- Kolone: MC14015BCP (dvostruki 4-bitni pomerački registar, **bez izlaznog
  latch-a**), jedan čip na 8 kolona; svaka kolona ima svoj TO-92 tranzistor.
- 5 lemnih tačaka na kraju modula — verovatno ulaz lanca (VDD, GND, DATA,
  CLK, RESET?), nije potvrđeno.
- Šema je dakle multipleks redova 1/8: kroz lanac 4015 se upiše red, pa se
  uključi njegov TO-220. Pošto 4015 nema latch, redovi moraju biti ugašeni
  dok traje pomeranje.

**Podaci od korisnika:**
- Panel je **64×8**: 8× MC14015 (po 8 kolona) = 64 kolone.
- Kolone: 64× **BC212** (PNP, ~100 mA) — prekidač kolone na plus strani.
- Redovi: 8× **2N6388** (NPN Darlington, TO-220, 10 A) — verovatno na minus
  strani (emiter na GND), nije izmereno.
- Napajanje po uputstvu: **9 V ±1 V, max 1,2 A (~11 W)**. Pri punom redu
  to je oko 1,2 A / 64 ≈ 19 mA po LED-u u impulsu.
- PCB se crta u **EasyEDA**.

Otvoreno: napon napajanja 4015 na modulu (pin 16 prema pinu 8). Ako je 9 V,
VIH je ~6,3 V, pa treba pomerač 3,3 → 9 V (npr. CD40109B ili MOSFET sa
pull-up-om), a ne 74HCT. Takođe otvoreno: raspored 5 lemnih tačaka na ulazu
modula (koristi li se RESET 4015), da li je emiter 2N6388 na GND, dimenzije
stare ploče.

Predlog za novu ploču (dok merenja ne potvrde):
- Redovi: 8× logic-level N-MOSFET (SOT-23, ~5 A, Vgs(th) ≤ 1,5 V), gejt
  direktno sa ESP32 preko ~100 Ω, pull-down 100 kΩ. Manji pad od Darlingtona.
- Napajanje: 9 V ulaz; LED struja ide direktno sa 9 V kroz BC212. Logika
  preko buck regulatora 9 → 5 V (ne LDO: 9 → 3,3 V pri ~0,5 A za WiFi je
  ~3 W toplote), ESP32 DevKit na 5 V, Schottky dioda da USB ne napaja 9 V
  granu.
- Pre pomeranja reda kroz 4015 ugasiti sve redove (nema latch-a).

**Odluka korisnika — nova PCB:** stara MCU ploča se izbacuje cela. Nova ploča
iste širine kao stara, da ulazi u iste žlebove kućišta. Na njoj: napajanje,
ESP32, prilagođavanje nivoa i drajveri redova; žice panela se prelemljuju na
nju. LED moduli ostaju netaknuti.

Plan:
1. Naći konektor između MCU ploče i LED modula, identifikovati signale
   (po mogućstvu logičkim analizatorom dok stara elektronika radi).
2. Prilagođavanje 3,3 V → 5 V (74HCT245 / 74AHCT125) — 4015 na 5 V traži
   VIH ≥ 3,5 V, pa ESP32 direktno nije dovoljan.
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
