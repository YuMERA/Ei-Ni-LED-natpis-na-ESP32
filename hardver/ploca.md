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
                                                            └─► J3 kolektori 2N6388

 ESP32 ─ SPI CLK/MOSI, RESET ─► U2 74AHCT125 (5 V) ─► J2 modul (+5V, GND, DATA, CLK, RESET)
 ESP32 ─ ROW1..8 ─► U3 74HCT245 (5 V) ─► 8× 220 Ω ─► J3 baze 2N6388
 ESP32 ─ ROW_OE ─► /OE U3 (pull-up: isključeno pri paljenju)
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

## ESP32

ESP32 DevKitC (38 pinova, WROOM-32) u ženskim letvicama.

| Signal | GPIO | Ide na |
|---|---|---|
| CLK | 18 (VSPI SCK) | U2 → J2 CLK |
| DATA | 23 (VSPI MOSI) | U2 → J2 DATA |
| RESET | 4 | U2 → J2 RESET |
| ROW_OE | 17 | /OE na U3 (0 = redovi uključeni) |
| ROW1–ROW8 | 13, 14, 25, 26, 27, 32, 33, 16 | U3 ulazi A1–A8 |
| (opciono) svetlo | 34 | foto-otpornik + 10 kΩ, automatska svetlina |

Izbegnuti su pinovi fleša (6–11) i strapping pinovi 0, 2, 5, 12, 15.

## Kolone — 74AHCT125 (U2)

MC14015 na modulu radi na **5 V** (izmerio korisnik: pin 16 je na izlazu
7805). Zato je dovoljan običan 3,3 → 5 V bafer; CD40109B nije potreban.

- U2 = 74AHCT125 (ili 74HCT125) na +5V; ulazi prihvataju 3,3 V sa ESP32.
- Kanali: DATA, CLK, RESET; četvrti neiskorišćen (ulaz na GND, /OE na +5V).
- /OE ulazi korišćenih kanala na GND (uvek aktivni).
- RESET: pull-down 100 kΩ na ESP32 strani.
- Opciono 33–100 Ω redno na CLK i DATA ka konektoru (dugačak kabl kroz
  ceo panel).
- 100 nF uz VCC.
- Pin +5V na J2 napaja 4015 na modulima — stari 7805 je bio na logičkoj
  ploči, pa sada to radi naš buck (struja je mala, CMOS).

## Redovi — 74HCT245 (U3) + postojeći 2N6388

Original (slika iz 2018): baze 2N6388 je vukao CDP1852 sa 5 V logike preko
**8× 220 Ω** (crvena-crvena-braon). 2N6388 je emiter-folower, pa je napon
reda ≈ 5 V − 2×Vbe ≈ **3,5 V**. Na modulu nema otpornika za LED — upravo taj
napon ograničava struju. **Baze se ne smeju vući na 9 V**: red bi dobio
~7,5 V i LED-ovi bi dobili višestruko veću struju.

- 2N6388 ostaju na aluminijumskom nosaču, kolektori na +9V, emiteri žutim
  žicama na redove modula (kao i sada). Grejanje je normalno: ~5 V pada na
  njima, zato je tu nosač.
- U3 = 74HCT245 na **+5V**, DIR na +5V (A → B). Izlazi B1–B8 preko 220 Ω
  na baze (isto kao original).
- Na svakoj bazi 10 kΩ ka GND — kad je U3 isključen (/OE = 1), izlazi
  plutaju, pa otpornik drži red ugašenim.
- /OE: 10 kΩ ka +5V → pri paljenju ESP32 svi redovi su ugašeni. Ovim se
  rešava problem da bi pri startu svih 8 redova moglo da se upali odjednom.
- Logika je prava: ROWx = 1 → red upaljen.
- ROW_OE istovremeno služi za gašenje dok se podaci pomeraju kroz 4015 i
  za PWM svetline.

## Konektori

| Oznaka | Pinovi | Tip |
|---|---|---|
| J1 | +9V, GND | DC utičnica + klema |
| J2 | +5V, GND, DATA, CLK, RESET | JST-XH 5-pin |
| J3 | +9V (kolektori), BAZA1–BAZA8 | JST-XH 9-pin (ili 10 sa GND); +9V nosi do 1,2 A |

Žice panela se kasnije rasporede u ženske konektore.

## Redosled u firmware-u (za jedan red)

1. ROW_OE = 1 (sve ugašeno).
2. Poslati 64 bita kroz SPI (~1 MHz → 64 µs).
3. ROW1–8: aktivni red = 1, ostali = 0.
4. ROW_OE = 0 (ili PWM) do sledećeg reda.

8 redova × 100 Hz = jedan red na 1,25 ms; pomeranje uzima ~5 % vremena.

## Šta još treba proveriti

- ~~VDD na 4015~~ — izmereno: 5 V sa 7805 na logičkoj ploči. Pošto je to
  jedini regulator, i CDP1852 koji je vukao baze redova radio je na 5 V —
  74HCT245 na 5 V + 220 Ω ponavlja isti napon reda.
- Polaritet kolona: da li BC212 provodi kad je izlaz 4015 nizak (verovatno)
  — rešava se u firmware-u, bez izmene ploče.
- Struja: na modulu nema otpornika za LED, pa prvo paliti uz izvor sa
  ograničenjem struje (npr. 9 V / 0,5 A) i meriti.
