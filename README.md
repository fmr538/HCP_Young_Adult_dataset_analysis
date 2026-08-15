# DWT koeficijenti kao obeležja fMRI signala (HCP Young Adult)

Seminarski rad iz predmeta Statistička analiza signala. Cilj je ispitati da li dodavanje
koeficijenata diskretne talasićne transformacije (DWT) skupu standardnih vremensko-frekvencijskih
obeležja poboljšava klasifikaciju sedam kognitivnih zadataka iz task-fMRI snimaka.

Rad ima tri odvojena nalaza. Prvi je da DWT koeficijenti ne poboljšavaju klasifikaciju, i to
zato što su **redundantni** sa standardnim skupom, a ne zato što su neinformativni. Drugi je da
prozori sidreni na fiksnu poziciju u runu **poravnavaju obeležja sa dizajnom paradigme**, što
sirovim vremenskim nizovima daje 52 procentna poena koje gube čim se poravnanje ukloni. Treći je
negativna replikacija modela Ertugrul i dr. (2018), sa izmerenim mehanizmom neuspeha.

## Podaci

**HCP Young Adult**, izdanje 2025 („ConnectomeDB powered by BALSA", reprocesirano u avgustu 2025),
grupa *100 Unrelated Subjects*. Preuzeto je **10 ispitanika**, paketi `Task3TRecommended`
i `StructuralRecommended`.

- 7 zadataka × 2 kodiranja faze (LR/RL) = **139 runova** (ispitanik `192237` nema `MOTOR_RL`)
- TR = 0.72 s; dužina runa zavisi od zadatka:
  EMOTION 176, GAMBLING 253, LANGUAGE 316, MOTOR 284, RELATIONAL 232, SOCIAL 274, WM 405 frejmova
- Koriste se volumetrijski fajlovi `*_hp0_clean_rclean_tclean.nii.gz` (91×109×91×T, 2 mm MNI)

Podaci **nisu** u repozitorijumu (110 GB) — vidi `.gitignore`. Očekivana putanja je
`data/<ispitanik>/MNINonLinear/`.

## Parcelacija

Signal se svodi na **87 regiona**: 68 kortikalnih parcela Desikan-Killiany atlasa
(kodovi 1001–1035 i 2001–2035) i 19 potkortikalnih struktura iz `aseg` segmentacije.

Oznake se čitaju iz `MNINonLinear/ROIs/wmparc.2.nii.gz`, koji je već na istoj rezoluciji i sa
istom afinom transformacijom kao funkcionalni snimci, pa nije potrebno preuzorkovanje.
Signal regiona je prosek voksela: `X_r(t) = (1/V_r) · Σ_{v∈r} X_v(t)`.

Napomena: izdanje iz 2025. ne isporučuje `aparc` na 32k površinskoj mreži, već samo u nativnoj
mreži i volumetrijski, zbog čega je analiza volumetrijska — što je ujedno i pristup rada [01].

## Prozori i podela

Svaki run se deli na prozore od **176 frejmova** (dužina najkraćeg runa) sa korakom 88, uz
završni prozor poravnat sa krajem runa → **357 prozora**. Fiksna dužina je neophodna jer dužina
runa jednoznačno određuje zadatak i inače bi curila u obeležja. Svaki prozor se z-normalizuje po
regionu; to je operacija unutar uzorka i ne curi kroz podelu.

Runovi različite dužine daju različit broj prozora, pa skup **nije uravnotežen**:

| WM | LANGUAGE | SOCIAL | MOTOR | GAMBLING | RELATIONAL | EMOTION |
|---|---|---|---|---|---|---|
| 80 | 60 | 60 | 57 | 40 | 40 | 20 |

Trivijalna granica je zato **većinska klasa, 0.224**, a ne uniformno pogađanje od 0.143. Sve
tabele u radu porede se sa 0.224.

Podela je po ispitanicima, nikada po prozorima (`seed = 0`):

| skup | ispitanici | uloga |
|---|---|---|
| train | 153126, 192237, 206525, 257946, 378756, 510225 | obučavanje |
| val | 111211, 869472 | svaki izbor hiperparametara |
| test | 135124, 590047 | jednokratna konačna ocena |

Uz to se sprovodi **leave-one-subject-out** (LOSO) preko svih 10 ispitanika: u svakom od 10
foldova model se obučava na devetorici i testira na desetom. LOSO daje procenu rasipanja po
ispitanicima, ali koristi sve ispitanike, pa **nije zamena za test skup** i ne sme da bira ništa.

## Obrada signala

Fajlovi nose oznaku `hp0`, dakle bez visokopropusnog filtriranja preko uklanjanja srednje
vrednosti. Provera pokazuje da dodatna obrada nije potrebna:

- udeo varijanse linearnog trenda: **0.000 ± 0.000** (HCP-ova obrada ga je već uklonila)
- udeo varijanse globalnog signala: **0.058 ± 0.020**

Ablacija koraka obrade (detrend, visokopropusni filtar, uklanjanje globalnog signala) ne daje
nijedno značajno poboljšanje; najveći kandidat je uklanjanje globalnog signala sa +0.040
(p = 0.398). Zaključak je da je HCP-ov cevovod dovoljan i da se dalje ne obrađuje.

Pretraga praga visokopropusnog filtra pokazuje **da najsporiji opseg nosi signal, a ne artefakt**:

| prag (Hz) | udeo A4 snage | F(band_A4) | LOSO |
|---|---|---|---|
| bez filtra | 0.263 | 7.6 | 0.673 |
| 0.01 | 0.236 | 7.9 | 0.679 |
| 0.02 | 0.143 | 7.5 | 0.629 |
| 0.04 | 0.030 | 4.7 | 0.515 |
| 0.08 | 0.001 | 3.4 | 0.468 |

Sečenje opsega 0–0.043 Hz košta dvadeset procentnih poena. Pošto linearnog trenda nema, ta
informacija nije drift skenera. Prag od 0.01 Hz je ispod frekvencijske rezolucije prozora
(1/126.7 s = 0.0079 Hz) i zato bez dejstva.

## Obeležja

**Standardna obeležja** (skup A, 9 tipova po regionu, 783 ukupno): relativna snaga u pet diadskih
opsega (granice na `f_Nyq / 2^k`, iste kao granice DWT podopsega), spektralna entropija,
spektralno težište, autokorelacija na kašnjenju 1 i 5.

**DWT obeležja** (870): `db4`, `L = 4` (maksimum za prozor od 176 frejmova), po podopsegu
relativna energija `E` i entropija koeficijenata `H`.

| Podopseg | Koeficijenata | Opseg |
|---|---|---|
| A4 | 17 | 0.000–0.043 Hz |
| D4 | 17 | 0.043–0.087 Hz |
| D3 | 28 | 0.087–0.174 Hz |
| D2 | 49 | 0.174–0.347 Hz |
| D1 | 91 | 0.347–0.694 Hz |

Skup B je A + DWT (1653 obeležja), skup C samo DWT (870).

**Klasifikatori.** Naivni Bayes (osnovni, kako predviđa postavka) i logistička regresija
(klasifikator iz rada [01]); regularizacija `C = 0.01` izabrana je na validacionom skupu, i to
isključivo nad skupom A, da poređenje A↔B ne dobije prednost od dvostrukog podešavanja.

## Analiza obeležja

**Redundantnost.** Granice opsega u standardnom skupu su namerno diadske, iste kao DWT podopsezi,
pa `band_s` (relativna snaga iz Welch-ove ocene) i `E_s` (relativna energija koeficijenata) mere
istu fizičku veličinu na dva načina:

| Podopseg | r(band, E) | F(band) | F(E) |
|---|---|---|---|
| A4 | 0.520 | 7.6 | 4.5 |
| D4 | 0.521 | 4.8 | 3.8 |
| D3 | 0.515 | 2.8 | 2.9 |
| D2 | 0.548 | 3.7 | 3.7 |
| D1 | 0.660 | 5.6 | 4.5 |

DWT energija nije duplikat, nego **lošija procena iste veličine** — `F(band) ≥ F(E)` u svakom
podopsegu. To je objašnjenje nultog rezultata: dodaje se šumnija mera nečega što se već meri bolje.

**Diskriminativnost po tipu obeležja** (prosečna F-vrednost, samo trening skup):

| entropy | ac5 | ac1 | centroid | band | E | H |
|---|---|---|---|---|---|---|
| 7.4 | 7.3 | 7.1 | 6.4 | 4.9 | 3.9 | **1.5** |

Entropija koeficijenata `H` — 435 od 870 DWT obeležja — nije izabrana **nijednom**, ni pri jednom
budžetu obeležja. Polovina DWT bloka ne doprinosi ništa.

**Regioni.** Najinformativniji su obostrani `lateraloccipital` i `fusiform`, zatim `supramarginal`,
`postcentral` i `precentral`. To odgovara skupu zadataka koji se najviše razlikuju po vizuelnoj
složenosti stimulusa, uz motorni doprinos iz zadatka MOTOR.

## Rezultati

Naivni Bayes, tri skupa obeležja:

| Skup | Obeležja | val | test | LOSO |
|---|---|---|---|---|
| A — standardna | 783 | 0.611 | **0.681** | 0.673 ± 0.228 |
| B — standardna + DWT | 1653 | 0.639 | 0.639 | 0.684 ± 0.229 |
| C — samo DWT | 870 | 0.528 | 0.583 | 0.582 ± 0.189 |
| većinska klasa | | | 0.224 | |

Po test ispitaniku: A daje 0.833 i 0.528, B daje 0.806 i 0.472 — dakle **oba** test ispitanika
rangiraju A iznad B. Logistička regresija daje suprotan smer (A: test 0.750, LOSO 0.724 ± 0.190;
B: test 0.778, LOSO 0.751 ± 0.219, razlika +0.027, p = 0.211). Efekat DWT-a je manji od razlike
među klasifikatorima.

Uparena poređenja po ispitaniku (Wilcoxon, Naivni Bayes):

| Poređenje | Razlika | p |
|---|---|---|
| B − A | +0.011 ± 0.053 | 0.727 |
| bez opsega snage + DWT − bez opsega snage | +0.017 ± 0.054 | 0.469 |
| vremenska lokalizacija koeficijenata − A | +0.003 ± 0.042 | 0.906 |
| B − A (logistička regresija) | +0.027 ± 0.056 | 0.211 |

**Dobitak postoji samo u većinskoj klasi.** Po zadatku (B − A): WM +0.112, SOCIAL +0.050,
LANGUAGE 0.000, MOTOR −0.018, GAMBLING −0.050, RELATIONAL −0.075, EMOTION −0.100. Ukupna tačnost
raste sa 0.678 na 0.689, a **makro F1 pada sa 0.654 na 0.653**. Onih +0.011 dolazi isključivo od
klase WM, koja ima 80 od 357 prozora.

**Redundantnost šteti pri ograničenom broju obeležja.** Selekcija po F-vrednosti unutar svakog
folda daje:

| k | A | B | B − A | p |
|---|---|---|---|---|
| 25 | 0.380 | 0.388 | +0.009 | 0.531 |
| 50 | 0.610 | 0.525 | **−0.085** | **0.018** |
| 100 | 0.674 | 0.638 | −0.036 | 0.141 |
| 200 | 0.690 | 0.685 | −0.005 | 0.930 |
| 400 | 0.682 | 0.715 | +0.033 | 0.141 |
| svi | 0.673 | 0.684 | +0.011 | 0.727 |

Pri k = 50 `E` zauzima 20% budžeta, ali samo 6 poena uzima od `band` (bezbolna zamena duplikata);
preostalih 14 poena uzima od `ac5`, `entropy` i `ac1`, dakle od tipova koji mere nešto drugo.
Odatle značajan gubitak.

## Izbor talasića

Rad [01] koristi kubne Battle-Lemarié talasiće, kojih u `pywt` nema. Poređenje kompaktno nosivih
porodica pokazuje da izbor ne utiče: db2 0.673, db4 0.684, db8 0.670, sym4 0.662, sym8 0.645,
coif2 0.654 — raspon 0.04 pri rasipanju po ispitanicima od ±0.22.

Battle-Lemarié je ipak sproveden **tačno**, zaobilaženjem filtarske banke. Filtar je beskonačan sa
eksponencijalnim opadanjem, pa bi skraćivanje na dužinu koja dopušta `L = 4` nosilo grešku
rekonstrukcije od 30% standardne devijacije. Pošto se koristi samo relativna energija po
podopsegu, po Parsevalovoj jednakosti je dovoljno `Σ_ω |X(ω)|²·|filtar(ω)|²`, a frekvencijski
odzivi ortonormalizovanog splajna se znaju u zatvorenom obliku. Potpunost banke je potvrđena na
`max |Σ|filtar|² − 1| = 9·10⁻⁸`.

Rezultat je da `db4` **precenjuje najniži opseg za 44%** (udeo A4: 0.390 naspram tačnih 0.270) i
potcenjuje D1 (0.230 naspram 0.288). Ipak, klasifikacija se ne menja: skup sa tačnim
Battle-Lemarié obeležjima daje 0.670 naspram 0.684 sa `db4`. Pristrasnost procene nije ono što
određuje rezultat.

## Poravnanje sa dizajnom

Prozori sidreni na fiksne pozicije u runu imaju isti raspored blokova kod svih ispitanika koji
rade isti zadatak. Kontrola je nasumičan pomak startne pozicije, uz isti broj prozora po runu
(dakle isto n i isti odnos klasa), u tri ponavljanja:

| Skup obeležja | poravnato | pomereno | pad |
|---|---|---|---|
| sirovi vremenski nizovi | 0.912 | 0.392 ± 0.006 | **−0.520** |
| parne korelacije | 0.806 | 0.799 ± 0.019 | −0.006 |
| mesh arc weights | 0.781 | 0.782 ± 0.008 | +0.001 |
| A — standardna | 0.673 | 0.670 ± 0.006 | −0.003 |
| B — standardna + DWT | 0.684 | 0.677 ± 0.009 | −0.007 |

Sve osim sirovog signala je nezavisno od položaja prozora, kako i sledi iz konstrukcije: snaga po
opsegu, entropija, težište, autokorelacija i korelacija među regionima ne zavise od toga gde
prozor počinje, dok vektor sirovih odbiraka zavisi.

Dve posledice. Glavni nalaz rada je **čist** — poređenje A naspram B nije zaraženo poravnanjem
(+0.011 na poravnatim, +0.007 na pomerenim prozorima). Vrednost od 0.912 za sirove nizove je
artefakt nacrta i mora se prijavljivati sa ispravkom na 0.392; time se vraća kvalitativni poredak
iz rada [01], `povezanost ≫ sirovi nizovi`. Napomena: EMOTION run ima tačno 176 frejmova, pa se
tih 20 prozora ne mogu pomeriti, i izmereni padovi su donja granica.

## Replikacija: Hierarchical Multi-resolution Mesh Networks [01]

Model iz rada [01] je implementiran u celini: rekonstrukcija 2L+1 = 9 podopsega inverznom DWT,
mesh mreža po podopsegu (p funkcionalno najbližih suseda, lučne težine iz rebraste regresije),
i fuzija odluka pod fuzzy stacked generalization (FSG) arhitekturom.

Odstupanja od rada, nametnuta podacima: L = 4 umesto 11 (prozor od 176 umesto 1940 frejmova),
`db4` umesto Battle-Lemarié u filtarskoj banci, 10 ispitanika umesto 808, 87 DK regiona umesto
90 AAL, i z-normalizacija podopsega da λ znači isto u svim opsezima.

**Poredak reprezentacija se ne replicira.** Rad prijavljuje mesh 97.15% > korelacije 89.97%; ovde
je na A0 mesh 0.781, korelacije 0.806. Posle fuzije preko svih podopsega mesh dostiže 0.817 ±
0.153, a korelacije 0.868 — mesh nigde ne vodi.

**Spektralni deo se replicira.** Tačnost detaljnih delova raste monotono sa nivoom (D1 0.241,
D2 0.293, D3 0.539, D4 0.651), što je uzlazna grana luka koji u radu ima vrh na l = 5–6; D1 je na
nivou većinske klase, što je razlog zbog kog ga rad odbacuje.

**Mehanizam neuspeha je izmeren.** Optimum po λ leži tačno na granici λ→∞, u kojoj lučna težina
*jeste* korelacija na p suseda (p = 10: 0.806 pri λ = 512, 0.806 pri λ→∞) — najbolja verzija mesh
modela je ona u kojoj prestaje da bude regresija. Hipoteza da je uzrok varijansa ocene je
oborena: razlika mesh−korel se sa dužim prozorom **širi**, a ne zatvara (λ = 32: −0.031 pri
TW = 88, −0.067 pri 132, −0.103 pri 176).

Uzrok je kolinearnost funkcionalnog susedstva. Uslovljenost matrice `C_ss` je ravna preko dužine
prozora (medijana 31.3 / 29.5 / 31.5 za TW 88 / 132 / 176), dok fazno randomizovana kontrola —
koja čuva spektar svakog regiona a razbija spregu među njima — pada za 39% (7.2 / 5.3 / 4.4).
Kolinearnost je dakle svojstvo populacije, a ne uzorka, i uzorkom se ne popravlja.

Uslovljenost ipak **nije usko grlo**. Pravila izbora suseda koja je prisilno popravljaju pogoršavaju
tačnost, monotono:

| Pravilo | cond | LOSO | mesh − korel | p |
|---|---|---|---|---|
| p najkorelisanijih (rad) | 29.1 | 0.781 | −0.024 | 0.242 |
| ortogonalno uparivanje | 27.4 | 0.787 | −0.019 | 0.533 |
| kolinearnost ≤ 0.6 | 11.2 | 0.731 | −0.075 | 0.004 |
| kolinearnost ≤ 0.5 | 7.9 | 0.716 | −0.090 | 0.029 |
| kolinearnost ≤ 0.4 | 5.5 | 0.677 | −0.129 | 0.006 |

Uslovljenost se popravlja pet puta, a tačnost monotono pada — jer se kupuje izbacivanjem upravo
onih suseda koji nose informaciju. Nijedno pravilo ne dovodi mesh iznad parnih korelacija.

## Zaključci

1. **DWT koeficijenti ne poboljšavaju klasifikaciju**, ni pri jednom od tri skupa obeležja, ni pri
   jednom od dva klasifikatora, ni pri jednom od šest isprobanih talasića. Razlog nije
   neinformativnost nego redundantnost: granice diadskih podopsega poklapaju se sa granicama
   opsega u kojima se već meri relativna snaga, a DWT je pri tome lošija procena iste veličine.
   Pri ograničenom broju obeležja redundantnost **aktivno šteti** (−0.085, p = 0.018 pri k = 50).

2. **Prividni dobitak od +0.011 nestaje kad se klase izjednače** — makro F1 je 0.654 naspram 0.653.
   Dobitak dolazi isključivo od najbrojnije klase.

3. **Poravnanje prozora sa dizajnom paradigme je ozbiljna zamka.** Sirovi vremenski nizovi gube
   52 procentna poena kad se poravnanje ukloni; obeležja korišćena u ovom radu ne gube ništa.

4. **Mesh mreže iz rada [01] ne nose informaciju preko parnih korelacija** na ovim podacima, ni
   pri jednom λ, ni pri jednoj dužini prozora, ni pri jednom pravilu izbora susedstva. Njihova
   najbolja radna tačka je granica u kojoj se svode na korelaciju.

## Ograničenja

Sa 10 ispitanika i rasipanjem razlika reda 0.05–0.08, najmanji efekat koji ovaj nacrt može da
detektuje je oko **7 procentnih poena** za poređenje A naspram B, odnosno 0.066 za mesh naspram
korelacija. Tvrdnje se zato pišu kao *efekat je ispod granice detekcije ovog nacrta*, a ne kao
*efekta nema*. Prednost mesh mreža nad korelacijama iz rada [01] iznosi 0.072 i leži tik iznad te
granice; izmereno je −0.024.

Test skup od dva ispitanika daje jedan broj bez procene rasipanja, pa se uz njega uvek navodi i
tačnost po ispitaniku. Dvadeset dodatnih ispitanika spustilo bi granicu detekcije na oko 0.05 i
učinilo centralno poređenje odlučivim; sve ostalo u radu je izmereno.

## Pokretanje

```bash
uv sync
uv run python -m ipykernel install --user --name hcp --display-name "Python (hcp)"
```

Zatim otvoriti `Projekat.ipynb` i izabrati kernel *Python (hcp)*. Sveska se izvršava odozgo
nadole; učitavanje i parcelacija svih 139 runova traje 30–60 minuta, a ceo prolaz oko dva sata.

## Struktura

```
Projekat.ipynb      cela analiza
literatura/         referentni radovi i uporedne tabele (nije u repou)
data/               HCP podaci (nije u repou)
pyproject.toml      zavisnosti (uv)
```

## Literatura

Detaljne beleške i uporedne tabele su u `literatura/README.md`.

1. Ertugrul, Ozay & Yarman-Vural (2018), *Hierarchical multi-resolution mesh networks for brain decoding* — ključni rad
2. Wang i dr. (2020), *Decoding and mapping task states of the human brain via deep learning*
3. Saeidi i dr. (2022), *Decoding Task-Based fMRI Data with Graph Neural Networks*
4. Bryant i dr. (2024), *Extracting interpretable signatures of whole-brain dynamics*
5. Kirova i dr. (2025), *Dynamic Functional Connectivity Features for Brain State Classification*
