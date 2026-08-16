# DWT koeficijenti kao obeležja fMRI signala (HCP Young Adult)

Seminarski rad iz predmeta Statistička analiza signala. Cilj je ispitati da li dodavanje
koeficijenata diskretne talasićne transformacije (DWT) skupu standardnih vremensko-frekvencijskih
obeležja poboljšava klasifikaciju sedam kognitivnih zadataka iz task-fMRI snimaka.

Rad ima tri nalaza. Prvi je da DWT koeficijenti **poboljšavaju klasifikaciju za oko dve procentne
tačke** pod logističkom regresijom (+0.023, p = 0.005, preživljava korekciju za višestruko
testiranje), a da je dobitak toliko mali zato što su **redundantni** sa standardnim skupom, a ne
zato što su neinformativni. Drugi je da prozori sidreni na fiksnu poziciju u runu **poravnavaju
obeležja sa dizajnom paradigme**, što sirovim vremenskim nizovima daje 51 procentni poen koje
gube čim se poravnanje ukloni. Treći je **uslovna replikacija** modela Ertugrul i dr. (2018):
mesh mreže nadmašuju parne korelacije, ali samo pod pravilom izbora susedstva boljim od onog koje
rad koristi, uz izmeren mehanizam.

## Podaci

**HCP Young Adult**, izdanje 2025 („ConnectomeDB powered by BALSA", reprocesirano u avgustu 2025),
grupa *100 Unrelated Subjects*. Preuzeto je **30 ispitanika**, paketi `Task3TRecommended`
i `StructuralRecommended`. Oba su neophodna: parcelacija čita `MNINonLinear/ROIs/wmparc.2.nii.gz`,
koji se isporučuje samo u strukturnom paketu.

- 7 zadataka × 2 kodiranja faze (LR/RL) = **419 runova** (ispitanik `192237` nema `MOTOR_RL`)
- TR = 0.72 s; dužina runa zavisi od zadatka:
  EMOTION 176, GAMBLING 253, LANGUAGE 316, MOTOR 284, RELATIONAL 232, SOCIAL 274, WM 405 frejmova
- Koriste se volumetrijski fajlovi `*_hp0_clean_rclean_tclean.nii.gz` (91×109×91×T, 2 mm MNI)

Podaci **nisu** u repozitorijumu (334 GB) — vidi `.gitignore`. Očekivana putanja je
`data/<ispitanik>/MNINonLinear/`. Rezultat parcelacije se kešira u `data/_cache/regions.npz`
(32 MB), pa se ponovno pokretanje sveske ne plaća ponovnim čitanjem svih runova.

## Parcelacija

Signal se svodi na **87 regiona**: 68 kortikalnih parcela Desikan-Killiany atlasa
(kodovi 1001–1035 i 2001–2035) i 19 potkortikalnih struktura iz `aseg` segmentacije.

Oznake se čitaju iz `MNINonLinear/ROIs/wmparc.2.nii.gz`, koji je već na istoj rezoluciji i sa
istom afinom transformacijom kao funkcionalni snimci, pa nije potrebno preuzorkovanje.
Parcelacija je po ispitaniku, dakle svaki ispitanik se deli sopstvenom `wmparc` mapom, a ne
zajedničkom grupnom. Signal regiona je prosek voksela: `X_r(t) = (1/V_r) · Σ_{v∈r} X_v(t)`.

Napomena: izdanje iz 2025. ne isporučuje `aparc` na 32k površinskoj mreži, već samo u nativnoj
mreži i volumetrijski, zbog čega je analiza volumetrijska — što je ujedno i pristup rada [01].

## Prozori i podela

Svaki run se deli na prozore od **176 frejmova** sa korakom 88, uz završni prozor poravnat sa
krajem runa → **1077 prozora**. Svaki prozor se z-normalizuje po regionu; to je operacija unutar
uzorka i ne curi kroz podelu.

Dužina prozora **nije slobodan parametar**. Odozgo je ograničena najkraćim runom (EMOTION, 176
frejmova), jer bi duži prozor tu klasu ostavio bez ijednog uzorka. Broj DWT nivoa je stepenasta
funkcija dužine, `L = ⌊log₂(W/7)⌋` za `db4`, pa svako `W ∈ [112, 223]` daje isti `L = 4`; peti
nivo bi tražio `W ≥ 224`. Frekvencijska rezolucija je `Δf = 1/(W·TR) = 0.0079 Hz`, tako da u
najniži opseg A4 (do 0.043 Hz) pada svega 5.5 Welch-ovih odbiraka — zbog čega se prozor ne sme
skraćivati. Empirijski, tačnost raste monotono do plafona (parne korelacije: 0.856 pri TW = 88,
0.868 pri 132, **0.894 pri 176**), iako se broj prozora pritom više nego prepolovljuje.

Fiksna dužina je uz to neophodna jer dužina runa jednoznačno određuje zadatak i inače bi curila
u obeležja.

Runovi različite dužine daju različit broj prozora, pa skup **nije uravnotežen**:

| WM | LANGUAGE | SOCIAL | MOTOR | GAMBLING | RELATIONAL | EMOTION |
|---|---|---|---|---|---|---|
| 240 | 180 | 180 | 177 | 120 | 120 | 60 |

Trivijalna granica je zato **većinska klasa, 0.223**, a ne uniformno pogađanje od 0.143. Sve
tabele u radu porede se sa 0.223.

Podela je po ispitanicima, nikada po prozorima (`seed = 0`):

| skup | ispitanici | prozora | uloga |
|---|---|---|---|
| train | 111211, 125424, 130518, 135124, 146735, 153126, 176845, 180230, 186848, 213017, 257946, 274542, 378756, 510225, 590047, 692964, 869472, 911849 | 648 | obučavanje |
| val | 137532, 151930, 192237, 199352, 227533, 300719 | 213 | svaki izbor hiperparametara |
| test | 118831, 165436, 206525, 392447, 519647, 698168 | 216 | jednokratna konačna ocena |

Uz to se sprovodi **leave-one-subject-out** (LOSO) preko svih 30 ispitanika. LOSO daje procenu
rasipanja po ispitanicima, ali koristi sve ispitanike, pa **nije zamena za test skup** i ne sme
da bira ništa.

## Obrada signala

Fajlovi nose oznaku `hp0`, dakle bez visokopropusnog filtriranja preko uklanjanja srednje
vrednosti. Merenja pokazuju:

- udeo varijanse linearnog trenda: **0.000 ± 0.000** (HCP-ova obrada ga je već uklonila)
- udeo varijanse globalnog signala: **0.056 ± 0.018**

Ablacija koraka obrade (LOSO, skup A):

| obrada | val | LOSO | razlika naspram neobrađenog | p |
|---|---|---|---|---|
| kako jeste | 0.737 | 0.734 ± 0.210 | — | — |
| detrend | 0.737 | 0.734 ± 0.210 | +0.000 ± 0.000 | 1.000 |
| detrend + hp 0.01 | 0.761 | 0.771 ± 0.158 | +0.038 ± 0.090 (15/30) | 0.109 |
| + uklanjanje globalnog signala | 0.751 | 0.747 ± 0.172 | +0.014 ± 0.110 | 0.749 |
| samo globalni signal | 0.737 | 0.722 ± 0.224 | −0.011 ± 0.062 | 0.397 |

**Nijedan korak ne daje pouzdano poboljšanje**, pa se dalja obrada ne sprovodi i svi rezultati
niže računati su nad signalom kakav HCP isporučuje. Visokopropusni filtar podiže prosek za 0.038,
ali polovina ispitanika ide u minus (15/30) i rasipanje je 0.090, dakle dvostruko veće od efekta.

Metodološka napomena: ranija verzija ove ablacije merila je `GroupKFold` sa 10 foldova po tri
ispitanika i za taj korak dobijala p = 0.004. Prelazak na LOSO, koji koristi ostatak rada, efekat
poništava. Prijavljuje se LOSO vrednost, jer bi mešanje dve šeme foldova činilo BH tabelu
nekonzistentnom.

Pretraga praga visokopropusnog filtra pokazuje **da najsporiji opseg nosi signal, a ne artefakt**:

| prag (Hz) | val | LOSO | F(band_A4) | udeo A4 snage |
|---|---|---|---|---|
| bez filtra | 0.737 | 0.734 ± 0.210 | 22.7 | 0.269 |
| 0.01 | 0.761 | 0.771 ± 0.158 | 24.6 | 0.241 |
| 0.02 | 0.690 | 0.709 ± 0.179 | 22.8 | 0.147 |
| 0.04 | 0.624 | 0.581 ± 0.211 | 15.1 | 0.031 |
| 0.08 | 0.498 | 0.505 ± 0.220 | 8.6 | 0.001 |

Sečenje opsega 0–0.043 Hz košta preko dvadeset procentnih poena, a F-vrednost samih `band_A4`
obeležja pritom pada sa 22.7 na 8.6. Pošto linearnog trenda nema, ta informacija nije drift
skenera.

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
(klasifikator iz rada [01]); regularizacija `C = 10` izabrana je na validacionom skupu iz mreže
0.003–100, sa unutrašnjim maksimumom (0.812 pri C = 3, **0.817 pri C = 10**, 0.808 pri 30 i 100),
i to isključivo nad skupom A, da poređenje A↔B ne dobije prednost od dvostrukog podešavanja.

## Analiza obeležja

**Redundantnost.** Granice opsega u standardnom skupu su namerno diadske, iste kao DWT podopsezi,
pa `band_s` (relativna snaga iz Welch-ove ocene) i `E_s` (relativna energija koeficijenata) mere
istu fizičku veličinu na dva načina:

| Podopseg | r(band, E) | F(band) | F(E) |
|---|---|---|---|
| A4 | 0.509 | 22.7 | 12.6 |
| D4 | 0.532 | 11.5 | 7.7 |
| D3 | 0.512 | 6.2 | 6.8 |
| D2 | 0.555 | 11.2 | 10.2 |
| D1 | 0.670 | 14.3 | 12.1 |

DWT energija nije duplikat, nego **lošija procena iste veličine** — `F(band) > F(E)` u četiri od
pet podopsega, sa izuzetkom D3, gde su praktično izjednačene. To je objašnjenje zašto je dobitak
mali: dodaje se šumnija mera nečega što se već meri bolje.

**Diskriminativnost po tipu obeležja** (prosečna F-vrednost, samo trening skup):

| entropy | ac5 | ac1 | centroid | band | E | H |
|---|---|---|---|---|---|---|
| 21.6 | 19.9 | 19.4 | 18.3 | 13.2 | 9.9 | **1.9** |

Entropija koeficijenata `H` — 435 od 870 DWT obeležja — ne ulazi u prvih 200 obeležja po
F-vrednosti, a pri budžetu od 400 zauzima svega 1% izabranog skupa. Polovina DWT bloka je
praktično neupotrebljena.

**Regioni.** Najinformativniji su obostrani `lateraloccipital` i `fusiform`, zatim `pericalcarine`
i `lingual`, pa `superiorparietal` i `postcentral`. To odgovara skupu zadataka koji se najviše
razlikuju po vizuelnoj složenosti stimulusa, uz motorni doprinos iz zadatka MOTOR.

## Rezultati

Naivni Bayes, tri skupa obeležja:

| Skup | Obeležja | val | test | LOSO | opseg po ispitaniku |
|---|---|---|---|---|---|
| A — standardna | 783 | 0.737 | 0.727 | 0.734 ± 0.210 | 0.06–0.97 |
| B — standardna + DWT | 1653 | 0.756 | **0.759** | 0.750 ± 0.205 | 0.17–1.00 |
| C — samo DWT | 870 | 0.685 | 0.694 | 0.674 ± 0.190 | 0.17–0.97 |
| većinska klasa | | | 0.223 | | |

Logistička regresija daje isti smer i viši nivo:

| Skup | test | LOSO | opseg |
|---|---|---|---|
| A — standardna | 0.829 | 0.835 ± 0.139 | 0.36–0.97 |
| B — standardna + DWT | **0.847** | **0.859 ± 0.128** | 0.42–1.00 |

Po test ispitaniku B nadmašuje A kod **pet od šest**: 0.667/0.694, 1.000/0.944, 0.722/0.667,
0.917/0.889, 0.444/0.417, 0.806/0.750 (B/A pod Naivnim Bayesom). Izuzetak je `118831`.

Pošto jedna podela na 30 ispitanika i dalje nosi rasipanje, poređenje je ponovljeno preko
**20 nasumičnih podela**: A 0.739 ± 0.075, B 0.760 ± 0.066, razlika +0.020 ± 0.022 sa rasponom od
−0.009 do +0.070, a B je iznad A na testu u **16 od 20** podela.

Uparena poređenja po ispitaniku (Wilcoxon nad 30 LOSO foldova):

| Poređenje | Razlika | p |
|---|---|---|
| **B − A (logistička regresija)** | **+0.023 ± 0.039** (17/30) | **0.003** |
| bez opsega snage + DWT − bez opsega snage | +0.022 ± 0.058 (17/30) | 0.048 |
| vremenska lokalizacija koeficijenata − A | +0.019 ± 0.048 (17/30) | 0.080 |
| B − A (Naivni Bayes) | +0.016 ± 0.046 (15/30) | 0.078 |

Sva četiri poređenja su pozitivna i istog reda veličine, a poređenje pod logističkom regresijom
**preživljava korekciju za višestruko testiranje** (BH 0.013). Razlika između klasifikatora je
očekivana: Naivni Bayes pretpostavlja nezavisnost obeležja, a skup B ima 1653 međusobno
korelisana obeležja, pa mu redundantni blok smeta više nego regularizovanoj regresiji.

**Dobitak nije artefakt neuravnoteženih klasa.** Po zadatku (B − A): WM +0.062, GAMBLING +0.042,
MOTOR +0.034, EMOTION +0.017, LANGUAGE −0.006, SOCIAL −0.011, RELATIONAL −0.058. Ukupna tačnost
raste sa 0.735 na 0.751, a **makro F1 sa 0.713 na 0.739** — dakle jače od ukupne tačnosti, jer
najveći dobici padaju na GAMBLING i EMOTION, klase sa najmanje prozora.

**Redundantnost šteti pri ograničenom broju obeležja.** Selekcija po F-vrednosti unutar svakog
folda daje:

| k | A | B | B − A | p |
|---|---|---|---|---|
| 25 | 0.580 | 0.574 | −0.005 | 0.738 |
| 50 | 0.664 | 0.639 | **−0.025** | 0.066 |
| 100 | 0.711 | 0.729 | +0.018 | 0.104 |
| 200 | 0.721 | 0.726 | +0.005 | 0.698 |
| 400 | 0.737 | 0.743 | +0.006 | 0.624 |
| 800 | 0.734 | 0.751 | +0.017 | 0.082 |
| svi | 0.734 | 0.750 | +0.016 | 0.078 |

Pri k = 50 `E` zauzima 22% budžeta, a `band` pada sa 47% na 37%; ostatak se uzima od `entropy`,
`ac5` i `ac1`, dakle od tipova koji mere nešto drugo. Efekat ne preživljava korekciju (BH 0.087),
pa se piše kao tendencija potkrepljena sastavom izabranog skupa, a ne kao izmeren efekat.

## Izbor talasića

Rad [01] koristi kubne Battle-Lemarié talasiće, kojih u `pywt` nema. Poređenje kompaktno nosivih
porodica pokazuje da izbor ne utiče: db2 0.764, db4 0.750, sym4 0.751, sym8 0.749, coif2 0.747,
db8 0.746 — raspon 0.018 pri rasipanju po ispitanicima od ±0.21. `db4` je izabran unapred i
zadržan.

Battle-Lemarié je ipak sproveden **tačno**, zaobilaženjem filtarske banke. Filtar je beskonačan sa
eksponencijalnim opadanjem, pa bi skraćivanje na dužinu koja dopušta `L = 4` nosilo grešku
rekonstrukcije od 30% standardne devijacije. Pošto se koristi samo relativna energija po
podopsegu, po Parsevalovoj jednakosti je dovoljno `Σ_ω |X(ω)|²·|filtar(ω)|²`, a frekvencijski
odzivi ortonormalizovanog splajna se znaju u zatvorenom obliku. Potpunost banke je potvrđena na
`max |Σ|filtar|² − 1| = 9.08·10⁻⁸`.

Rezultat je da `db4` **precenjuje najniži opseg za 44%** (udeo A4: 0.397 naspram tačnih 0.276) i
potcenjuje D1 (0.224 naspram 0.282). Ipak, klasifikacija se ne menja: skup sa tačnim
Battle-Lemarié obeležjima daje 0.735 ± 0.212 naspram 0.750 ± 0.205 sa `db4`. Pristrasnost procene
nije ono što određuje rezultat.

## Poravnanje sa dizajnom

Prozori sidreni na fiksne pozicije u runu imaju isti raspored blokova kod svih ispitanika koji
rade isti zadatak. Kontrola je nasumičan pomak startne pozicije, uz isti broj prozora po runu
(dakle isto n i isti odnos klasa), u tri ponavljanja:

| Skup obeležja | poravnato | pomereno | pad |
|---|---|---|---|
| sirovi vremenski nizovi | 0.956 ± 0.052 | 0.448 ± 0.010 | **−0.508** |
| mesh arc weights | 0.907 ± 0.094 | 0.897 ± 0.004 | −0.010 |
| parne korelacije | 0.894 ± 0.113 | 0.884 ± 0.002 | −0.009 |
| B — standardna + DWT | 0.750 ± 0.205 | 0.741 ± 0.007 | −0.009 |
| A — standardna | 0.734 ± 0.210 | 0.714 ± 0.002 | −0.019 |

Rasipanje uz „poravnato" je po ispitanicima (LOSO), uz „pomereno" po ponavljanjima.

Sve osim sirovog signala je nezavisno od položaja prozora, kako i sledi iz konstrukcije: snaga po
opsegu, entropija, težište, autokorelacija i korelacija među regionima ne zavise od toga gde
prozor počinje, dok vektor sirovih odbiraka zavisi.

Dve posledice. Glavni nalaz rada je **čist** — poređenje A naspram B nije zaraženo poravnanjem
(+0.016 na poravnatim, +0.027 na pomerenim prozorima). Vrednost od 0.956 za sirove nizove je
artefakt nacrta i mora se prijavljivati sa ispravkom na 0.448; time se vraća kvalitativni poredak
iz rada [01], `povezanost ≫ sirovi nizovi`. Napomena: EMOTION run ima tačno 176 frejmova, pa se
tih 60 prozora ne mogu pomeriti, i izmereni padovi su donja granica.

## Replikacija: Hierarchical Multi-resolution Mesh Networks [01]

Model iz rada [01] je implementiran u celini: rekonstrukcija 2L+1 = 9 podopsega inverznom DWT,
mesh mreža po podopsegu (p funkcionalno najbližih suseda, lučne težine iz rebraste regresije),
i fuzija odluka pod fuzzy stacked generalization (FSG) arhitekturom.

Odstupanja od rada, nametnuta podacima: L = 4 umesto 11 (prozor od 176 umesto 1940 frejmova),
`db4` umesto Battle-Lemarié u filtarskoj banci, 30 ispitanika umesto 808, 87 DK regiona umesto
90 AAL, i z-normalizacija podopsega da λ znači isto u svim opsezima.

Broj suseda i regularizacija biraju se **na validacionom skupu**, nad mrežom p ∈ {10, 20, 40} i
λ ∈ {512, 2048, 8192, 32768, ∞}: izabrano `p = 20, λ = 512` (val 0.887; granica λ→∞ pri istom p
daje 0.836). Granica λ→∞, u kojoj lučna težina *jeste* korelacija na p suseda, računa se kao
dijagnostika ali se ne bira, jer u njoj mesh prestaje da bude regresija.

Tačnost po podopsegu i reprezentaciji (LOSO):

| podopseg | mesh | mesh-OMP | korelacije | sirovi nizovi |
|---|---|---|---|---|
| A0 | 0.907 | **0.919** | 0.894 | 0.956 |
| A1 | 0.906 | 0.918 | 0.884 | 0.961 |
| A2 | 0.910 | 0.910 | 0.892 | 0.961 |
| A3 | 0.889 | 0.893 | 0.891 | 0.953 |
| A4 | 0.863 | 0.876 | 0.868 | 0.892 |
| D4 | 0.751 | 0.784 | 0.781 | 0.777 |
| D3 | 0.650 | 0.650 | 0.671 | 0.711 |
| D2 | 0.354 | 0.346 | 0.369 | 0.441 |
| D1 | 0.260 | 0.260 | 0.287 | 0.208 |

**Poredak reprezentacija se replicira uslovno.** Rad prijavljuje mesh 97.15% > korelacije 89.97%.
Ovde je pod pravilom iz samog rada (p najkorelisanijih suseda) mesh 0.907 naspram 0.894, dakle
+0.013 i **nije značajno** (p = 0.452). Pod ortogonalnim uparivanjem mesh daje 0.919, odnosno
**+0.026 uz p = 0.016**, što preživljava BH korekciju; samo poređenje dva pravila daje
`OMP − top-p = +0.012, p = 0.043`. Zaključak je da rad ima pravo u pogledu poretka, ali sa
suboptimalnim pravilom izbora susedstva.

Posle fuzije preko svih podopsega mesh dostiže 0.924 (FSG-L, E = 9), mesh-OMP 0.934 (FSG-L,
E = 6), korelacije 0.908, sirovi nizovi 0.967. Konfiguracija iz rada (mesh + FSG-S, E = 9) daje
**0.910 ± 0.093**, naspram 0.9572 koliko rad prijavljuje.

Model je pritom znatno jači od standardnog skupa obeležja: HMMN − A `+0.176 ± 0.156` (25/30,
p < 0.001), HMMN − B `+0.160`, a i protiv logističke regresije nad istim obeležjima `+0.075`
(p = 0.001) odnosno `+0.051` (p = 0.010). Povezanost među regionima nosi informaciju koju
vremensko-frekvencijska obeležja po regionu ne hvataju.

**Spektralni deo se replicira.** Tačnost detaljnih delova raste monotono sa nivoom (D1 0.260,
D2 0.354, D3 0.650, D4 0.751), što je uzlazna grana luka koji u radu ima vrh na l = 5–6; D1 je na
nivou većinske klase, što je razlog zbog kog ga rad odbacuje.

**Uslovljenost nije usko grlo — izmereno u oba smera.** Pravila izbora suseda koja je prisilno
popravljaju pogoršavaju tačnost, monotono:

| Pravilo | cond | LOSO | mesh − korel | p |
|---|---|---|---|---|
| ortogonalno uparivanje | 40.7 | **0.919** | +0.026 | 0.016 |
| p najkorelisanijih (rad) | 63.0 | 0.907 | +0.013 | 0.452 |
| kolinearnost ≤ 0.6 | 21.0 | 0.873 | −0.021 | 0.017 |
| kolinearnost ≤ 0.5 | 14.3 | 0.860 | −0.034 | < 0.001 |
| kolinearnost ≤ 0.4 | 9.6 | 0.854 | −0.040 | < 0.001 |

Uslovljenost se popravlja skoro sedam puta, a tačnost pada za šest procentnih poena, jer se
kupuje izbacivanjem upravo onih suseda koji nose informaciju.

Regularizacija se ponaša isto: pri TW = 176 razlika mesh − korel ide od −0.082 (λ = 8, p < 0.001)
preko −0.040 (λ = 32) do −0.004 (λ = 128) i +0.005 (λ = 2048), dakle preterano prigušivanje šteti,
a optimum je pri slaboj regularizaciji.

Kolinearnost funkcionalnog susedstva je **svojstvo populacije, a ne uzorka**: uslovljenost matrice
`C_ss` je ravna preko dužine prozora (medijana 81.6 / 67.2 / 66.4 za TW 88 / 132 / 176), dok fazno
randomizovana kontrola — koja čuva spektar svakog regiona a razbija spregu među njima — pada na
18.6 / 11.7 / 9.0, uz pad prosečne apsolutne korelacije među susedima sa 0.287 na 0.107. Duži
prozor je dakle ne popravlja.

## Višestruko testiranje

Sveska sprovodi 14 uparenih poređenja nad istim podacima, pa se p-vrednosti koriguju po
Benjamini-Hochberg postupku:

| poređenje | razlika | p | p (BH) |
|---|---|---|---|
| HMMN − A (Naivni Bayes) | +0.176 | < 0.001 | **< 0.001** |
| mesh − korel (τ = 0.4) | −0.040 | < 0.001 | **0.001** |
| mesh − korel (τ = 0.5) | −0.034 | < 0.001 | **0.001** |
| HMMN − A (log. regresija) | +0.075 | 0.001 | **0.003** |
| **B − A (log. regresija)** | **+0.023** | **0.005** | **0.013** |
| mesh − korel (τ = 0.6) | −0.021 | 0.012 | **0.028** |
| **mesh − korel (OMP)** | **+0.026** | **0.014** | **0.028** |
| bez opsega + DWT − bez opsega | +0.022 | 0.054 | 0.087 |
| vremenska lokalizacija − A | +0.019 | 0.060 | 0.087 |
| B − A pri k = 50 | −0.025 | 0.062 | 0.087 |
| B − A (Naivni Bayes) | +0.016 | 0.114 | 0.145 |
| mesh − korel (top-p) | +0.013 | 0.385 | 0.449 |
| B − A pri k = 400 | +0.006 | 0.522 | 0.562 |
| globalni signal − bez obrade | +0.014 | 0.749 | 0.749 |

Sedam poređenja preživljava korekciju. Tri τ reda nisu tri nezavisna nalaza nego jedan, izmeren
na tri praga, pa BH postaje konzervativniji prema ostalima nego što bi bio da su sažeti u jedan.

## Zaključci

1. **DWT koeficijenti poboljšavaju klasifikaciju, ali malo.** Pod logističkom regresijom razlika
   je +0.023 (p = 0.005, BH 0.013), a efekat je pozitivan i u sva tri preostala poređenja
   (+0.016 do +0.022). Dobitak je mali zbog redundantnosti: granice diadskih podopsega poklapaju
   se sa granicama opsega u kojima se već meri relativna snaga, a DWT je pri tome lošija procena
   iste veličine (`F(band) > F(E)` u četiri od pet podopsega). Pod Naivnim Bayesom efekat ne
   dostiže značajnost jer klasifikator pretpostavlja nezavisnost obeležja.

2. **Dobitak nije artefakt neuravnoteženih klasa.** Makro F1 raste sa 0.713 na 0.739, dakle jače
   od ukupne tačnosti (0.735 → 0.751).

3. **Poravnanje prozora sa dizajnom paradigme je ozbiljna zamka.** Sirovi vremenski nizovi gube
   51 procentni poen kad se poravnanje ukloni; obeležja korišćena u ovom radu gube najviše dva.

4. **Mesh mreže nadmašuju parne korelacije, ali ne pod pravilom koje rad koristi.** Sa
   ortogonalnim uparivanjem +0.026 (BH 0.028); sa pravilom iz rada +0.013 i bez značajnosti.
   Uslovljenost nije uzrok: oba pravila bez ograničenja kolinearnosti (cond 40.7 i 63.0)
   nadmašuju sva tri koja je ograničavaju (cond 9.6 do 21.0), i to za do šest procentnih poena.

5. **Sam HMMN model je ubedljivo jači od standardnog skupa obeležja** (+0.176 pod Naivnim
   Bayesom, +0.075 protiv logističke regresije), i sa 0.910 se približava vrednosti od 0.9572 iz
   rada, uprkos L = 4 umesto 11 i 30 ispitanika umesto 808.

## Ograničenja

Najmanji efekat koji ovaj nacrt može da detektuje kreće se od **0.025 do 0.047**, zavisno od
rasipanja konkretnog poređenja. Tvrdnje o odsustvu efekta pišu se kao *efekat je ispod granice
detekcije ovog nacrta*, a ne kao *efekta nema*.

Nalaz `mesh − korel (OMP)` počiva na doslednosti znaka, ne na veličini: `+0.026 ± 0.082` uz n = 30
daje `t ≈ 1.7`, pa ga parametarski test ne bi proglasio značajnim, a i sam je manji od granice
detekcije za to poređenje (0.047). Wilcoxon prolazi jer je rangovni. Prijavljuje se kao značajan
uz ovu ogradu.

Poređenje `B − A` pod Naivnim Bayesom ostaje neodlučivo (+0.016, p = 0.114). Pošto je pod
logističkom regresijom isti efekat značajan, razlika je pripisana pretpostavci nezavisnosti, ali
to je objašnjenje, a ne merenje.

Test skup od šest ispitanika daje jedan broj sa skromnom procenom rasipanja, pa se uz njega uvek
navodi i tačnost po ispitaniku, kao i ponavljanje preko 20 nasumičnih podela.

## Pokretanje

```bash
uv sync
uv run python -m ipykernel install --user --name hcp --display-name "Python (hcp)"
```

Zatim otvoriti `Projekat.ipynb` i izabrati kernel *Python (hcp)*. Sveska se izvršava odozgo
nadole i **proverena je punim prolazom iz praznog kernela**. Prvo pokretanje parcelira svih 419
runova i traje oko dva sata; rezultat se kešira u `data/_cache/regions.npz`, pa naredni prolaz
kreće za nekoliko sekundi i traje oko dva sata, uglavnom zbog mesh mreža.

## Struktura

```
Projekat.ipynb          cela analiza
literatura/             referentni radovi i uporedne tabele (nije u repou)
data/                   HCP podaci (nije u repou)
data/_cache/            keširana parcelacija (nije u repou)
pyproject.toml          zavisnosti (uv)
```

## Literatura

Detaljne beleške i uporedne tabele su u `literatura/README.md`.

1. Ertugrul, Ozay & Yarman-Vural (2018), *Hierarchical multi-resolution mesh networks for brain decoding* — ključni rad
2. Wang i dr. (2020), *Decoding and mapping task states of the human brain via deep learning*
3. Saeidi i dr. (2022), *Decoding Task-Based fMRI Data with Graph Neural Networks*
4. Bryant i dr. (2024), *Extracting interpretable signatures of whole-brain dynamics*
5. Kirova i dr. (2025), *Dynamic Functional Connectivity Features for Brain State Classification*
