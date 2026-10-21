# Nova logička ploča — predlog šeme

Zamena za staru RCA 1802 ploču u panelu INFOS 1802 (64×8). Crta se u EasyEDA.
Ovo je predlog za crtanje; ništa još nije probano na panelu.

## Mere

- Širina **90 mm**, dužina po izboru (predlog ~100 mm). Ploča ulazi u žlebove
  za logiku.
- Debljina 1,6 mm.
- Uz obe duže ivice ostaviti **≥ 3 mm bez bakra i delova** — to klizi u
  vođice.
- USB konektor ESP32 DevKit-a okrenuti ka kraju kućišta koji se može otvoriti.

## Blok-šema

```
 9 V ─► J1 ─► Q1 (zaštita od obrnutog polariteta) ─► +9V ──┬─► C1 1000 µF
                                                            ├─► U1 buck 9→5 V ─► +5V ─► D1 ─► ESP32 5V
                                                            ├─► JP1 (VDD_MOD = 5 V ili 9 V)
                                                            └─► Q2 (P-MOSFET, ROW_EN) ─► +9V_ROW

 ESP32 ─ SPI CLK/MOSI, RESET ─► U2 CD40109B ─► J2 modul (VDD_MOD, GND, DATA, CLK, RESET)
 ESP32 ─ ROW1..8 ─► U3 ULN2803 ─► baze 2N6388 (pull-up 8× R na +9V_ROW) ─► J3
 ESP32 ─ ROW_EN ─► Q3 2N7002 ─► gejt Q2
```

## Napajanje

| Oznaka | Deo | Napomena |
|---|---|---|
| J1 | DC utičnica 5,5/2,1 mm + klema 2-pin paralelno | ulaz 9 V |
| F1 | PTC osigurač 2 A (1812) | |
| Q1 | AO3407A (P-MOSFET, −30 V, Vgs ±20 V) | zaštita od obrnutog polariteta, bez pada napona kao kod diode — bitno jer na modulu nema otpornika za LED |
| C1 | 1000 µF / 16 V elektrolit + 100 nF | impulsi reda do ~1,2 A |
| U1 | mini buck modul 9 → 5 V (MP1584EN, podesiv) ili AP63205WU na ploči | samo za logiku, ~0,5 A |
| D1 | B5819W (Schottky) | 5 V → pin 5V DevKit-a; USB sa računara ne napaja ostatak ploče |
| JP1 | 3-pin header + kratkospojnik | VDD_MOD = +5V ili +9V, prema izmerenom VDD na 4015 |

## ESP32

ESP32 DevKitC (38 pinova, WROOM-32) u ženskim letvicama. 3V3 sa DevKit-a
napaja CD40109B (strana VCC).

| Signal | GPIO | Ide na |
|---|---|---|
| CLK | 18 (VSPI SCK) | U2 → J2 CLK |
| DATA | 23 (VSPI MOSI) | U2 → J2 DATA |
| RESET | 4 | U2 → J2 RESET |
| ROW_EN | 17 | Q3 → Q2 |
| ROW1–ROW8 | 13, 14, 25, 26, 27, 32, 33, 16 | U3 ulazi 1–8 |
| (opciono) svetlo | 34 | foto-otpornik + 10 kΩ, automatska svetlina |

Izbegnuti su pinovi fleša (6–11) i strapping pinovi 0, 2, 5, 12, 15.

## Kolone — CD40109B (U2)

- Strana VCC = 3,3 V (ulazi A–C od ESP32), strana VDD = VDD_MOD (izlazi
  ka modulu). Ista ploča radi i za 5 V i za 9 V na modulu.
- Ulazi DATA, CLK i RESET: 3 kanala, četvrti neiskorišćen (ulaz na GND).
- Enable ulaze vezati tako da su izlazi uvek aktivni — proveriti u
  datasheet-u na koju stranu napajanja se odnose.
- RESET: pull-down 100 kΩ na ESP32 strani.
- 100 nF uz VCC i uz VDD.

## Redovi — ULN2803 (U3) + postojeći 2N6388

- 2N6388 ostaju na aluminijumskom nosaču, kolektori na +9V_ROW, emiteri
  žutim žicama na redove modula (kao i sada).
- Baza svakog 2N6388: otpornik ka +9V_ROW (pull-up, **predlog 4,7 kΩ**;
  ako se na panelu B mogu pročitati originalni otpornici R1–R11, uzeti te
  vrednosti) i izlaz ULN2803 ka masi.
- **Logika je obrnuta:** ROWx = 1 → ULN vuče bazu dole → red ugašen.
  Aktivan red je onaj čiji je pin 0.

## Glavni prekidač redova — Q2/Q3

Problem: pri uključenju ESP32 pinovi su 0 ili plutaju, ULN je isključen, pa
bi bez ovoga **svih 8 redova bilo upaljeno odjednom** sa nasumičnim
kolonama — 8× veća prosečna struja od predviđene.

- Q2: AO3407A (P-MOSFET), source na +9V, drain = +9V_ROW (kolektori i
  pull-up otpornici baza).
- Gejt Q2: 10 kΩ ka source-u (podrazumevano isključen), drain Q3 ga spušta.
- Q3: 2N7002, gejt sa ROW_EN preko 100 Ω, pull-down 100 kΩ.
- ROW_EN = 1 → redovi imaju napajanje. Isti pin služi za gašenje dok se
  podaci pomeraju kroz 4015 i za PWM svetline.

## Konektori

| Oznaka | Pinovi | Tip |
|---|---|---|
| J1 | +9V, GND | DC utičnica + klema |
| J2 | VDD_MOD, GND, DATA, CLK, RESET | JST-XH 5-pin |
| J3 | +9V_ROW, BAZA1–BAZA8 | JST-XH 9-pin (ili 10 sa GND) |

Žice panela se kasnije rasporede u ženske konektore.

## Redosled u firmware-u (za jedan red)

1. ROW_EN = 0 (sve ugašeno).
2. Poslati 64 bita kroz SPI (~1 MHz → 64 µs).
3. ROW1–8: aktivni red = 0, ostali = 1.
4. ROW_EN = 1 (ili PWM) do sledećeg reda.

8 redova × 100 Hz = jedan red na 1,25 ms; pomeranje uzima ~5 % vremena.

## Šta još treba proveriti

- VDD na 4015 (pin 16 prema pinu 8) na panelu B — određuje JP1.
- Polaritet kolona: da li BC212 provodi kad je izlaz 4015 nizak (verovatno)
  — rešava se u firmware-u, bez izmene ploče.
- Struja: na modulu nema otpornika za LED, pa prvo paliti uz izvor sa
  ograničenjem struje (npr. 9 V / 0,5 A) i meriti.
