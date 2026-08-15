# DWT koeficijenti kao obeležja fMRI signala (HCP Young Adult)

Seminarski rad iz predmeta Statistička analiza signala. Cilj je ispitati da li dodavanje
koeficijenata diskretne talasićne transformacije (DWT) skupu standardnih vremensko-frekvencijskih
obeležja poboljšava klasifikaciju sedam kognitivnih zadataka iz task-fMRI snimaka.

Slučajno pogađanje za sedam klasa iznosi **14.29%**.

## Podaci

**HCP Young Adult**, izdanje 2025 („ConnectomeDB powered by BALSA", reprocesirano u avgustu 2025),
grupa *100 Unrelated Subjects*. Preuzeto je **10 ispitanika**, paketi `Task3TRecommended`
i `StructuralRecommended`.

- 7 zadataka × 2 kodiranja faze (LR/RL) = **139 runova** (ispitanik `192237` nema `MOTOR_RL`)
- TR = 0.72 s; dužina runa zavisi od zadatka:
  EMOTION 176, GAMBLING 253, LANGUAGE 316, MOTOR 284, RELATIONAL 232, SOCIAL 274, WM 405 frejmova
- Koriste se volumetrijski fajlovi `*_hp0_clean_rclean_tclean.nii.gz` (91×109×91×T, 2 mm MNI)

Podaci **nisu** u repozitorijumu (111 GB) — vidi `.gitignore`. Očekivana putanja je
`data/<ispitanik>/MNINonLinear/`.

## Parcelacija

Signal se svodi na **87 regiona**: 68 kortikalnih parcela Desikan-Killiany atlasa
(kodovi 1001–1035 i 2001–2035) i 19 potkortikalnih struktura iz `aseg` segmentacije.

Oznake se čitaju iz `MNINonLinear/ROIs/wmparc.2.nii.gz`, koji je već na istoj rezoluciji i sa
istom afinom transformacijom kao funkcionalni snimci, pa nije potrebno preuzorkovanje.
Signal regiona je prosek voksela: `X_r(t) = (1/V_r) · Σ_{v∈r} X_v(t)`.

Napomena: izdanje iz 2025. ne isporučuje `aparc` na 32k površinskoj mreži, već samo u nativnoj
mreži i volumetrijski, zbog čega je analiza volumetrijska — što je ujedno i pristup rada [01].

## Metod

**Prozori.** Svaki run se deli na prozore od **176 frejmova** (dužina najkraćeg runa) sa korakom
88, uz završni prozor poravnat sa krajem runa → **357 prozora**. Fiksna dužina je neophodna jer
dužina runa jednoznačno određuje zadatak i inače bi curila u obeležja.

Svaki prozor se z-normalizuje po regionu (operacija unutar uzorka, ne curi kroz podelu).

**Podela.** Po ispitanicima, nikada po prozorima: 6 za trening, 2 za validaciju, 2 za test
(`seed = 0`). Dodatno se sprovodi **leave-one-subject-out** preko svih 10 ispitanika.

**Standardna obeležja** (9 po regionu, 783 ukupno): relativna snaga u pet diadskih opsega
(granice na `f_Nyq / 2^k`, isto kao DWT podopsezi), spektralna entropija, spektralno težište,
autokorelacija na kašnjenju 1 i 5.

**DWT obeležja** (870): `db4`, `L = 4` (maksimum za prozor od 176 frejmova), po podopsegu
relativna energija i entropija koeficijenata.

| Podopseg | Koeficijenata | Opseg |
|---|---|---|
| A4 | 17 | 0.000–0.043 Hz |
| D4 | 17 | 0.043–0.087 Hz |
| D3 | 28 | 0.087–0.174 Hz |
| D2 | 49 | 0.174–0.347 Hz |
| D1 | 91 | 0.347–0.694 Hz |

**Klasifikatori.** Naivni Bayes (osnovni, kako predviđa postavka) i logistička regresija
(klasifikator iz rada [01]).

## Rezultati (LOSO, tačnost)

| Skup obeležja | Obeležja | Naivni Bayes | Logistička regresija |
|---|---|---|---|
| A — standardna | 783 | 0.673 ± 0.228 | **0.732 ± 0.200** |
| B — standardna + DWT | 1653 | 0.684 ± 0.229 | **0.754 ± 0.225** |
| C — samo DWT | 870 | 0.582 ± 0.189 | — |
| bez opsega snage | 348 | 0.662 ± 0.218 | — |
| bez opsega snage + DWT | 1218 | 0.679 ± 0.211 | — |
| standardna + DWT (vremenska lokalizacija) | 2088 | 0.676 ± 0.224 | — |
| slučajno pogađanje | | 0.143 | |

Uparena poređenja po ispitaniku (Wilcoxon):

| Poređenje | Razlika | p |
|---|---|---|
| B − A (NB) | +0.011 ± 0.053 | 0.73 |
| bez opsega + DWT − bez opsega (NB) | +0.017 ± 0.054 | 0.47 |
| vremenska lokalizacija − A (NB) | +0.003 ± 0.042 | 0.91 |
| B − A (LR) | +0.022 ± 0.045 | 0.50 |

**Zaključak.** Dodavanje DWT koeficijenata ne poboljšava klasifikaciju značajno, ni pri jednoj od
tri isprobane formulacije obeležja, ni pri jednom od dva klasifikatora. Uz 10 ispitanika i
standardnu devijaciju razlika od približno 0.05, najmanji efekat koji ovaj nacrt može da detektuje
je oko **6 procentnih poena**; izmerene razlike su reda 0.3–2.2 poena. Tvrdnja je dakle da je
efekat manji od granice detekcije, a ne da ga nema.

Najveći pojedinačni efekat dolazi od izbora klasifikatora (+6 poena, NB → LR), a ne od obeležja.
Razlika među ispitanicima (0.12–0.97) višestruko nadmašuje sve razlike među skupovima obeležja.

## Pokretanje

```bash
uv sync
uv run python -m ipykernel install --user --name hcp --display-name "Python (hcp)"
```

Zatim otvoriti `Projekat.ipynb` i izabrati kernel *Python (hcp)*. Sveska se izvršava odozgo
nadole; učitavanje i parcelacija svih 139 runova traje 30–60 minuta.

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
