# CLAUDE.md — Pension-visualisering

## Projektbeskrivelse

Dette projekt er en **interaktiv HTML-visualisering af pensionsdepotudvikling** for danske pensionspolicer. Systemet fremregner depotværdier måned for måned og præsenterer resultatet som én selvstændig HTML-fil med grafer og interaktive parametre.

- **Output:** Én selvstændig `pension.html` (ingen build-step, ingen npm, kører direkte i browser)
- **Teknologi:** Vanilla JavaScript + [Chart.js](https://www.chartjs.org/) eller [Plotly.js](https://plotly.com/javascript/) via CDN
- **Formål:** Aktuarmæssig fremregningsmodel til illustration og indberetning (Finanstilsynet)
- **Sprog i kode og kommentarer:** Dansk

---

## De fire produkter

| Produkt | Type | Præmiefradrag | Skattefri udbetaling | Overlevelsesgevinster |
|---|---|---|---|---|
| Ratepension | Opsparing (10–30 år) | Ja (op til loft) | Nej | Nej |
| Livrente | Opsparing (livsvarig) | Ja (ubegrænset) | Nej | **Ja** |
| Aldersopsparing | Opsparing | Nej | **Ja** | Nej |
| Livsforsikring | Rent risikotillæg | — | — | — |

**Vigtig pointe:** Depot-mekanikken er **identisk** for alle tre opsparingsprodukter. Forskellen ligger i skattebehandling, konverteringsregler og adgang til overlevelsesgevinster.

Livsforsikring sælges altid som **tillæg** til et af de tre opsparingsprodukter og opbygger ikke depot.

---

## Notationstabel

| Symbol | Beskrivelse |
|---|---|
| `D_t` | Depotværdi ved starten af måned `t` |
| `π` | Månedlig bruttopræmie (fast indbetalingsbeløb) |
| `δ_liv,t` | Månedlig livsforsikringspræmie (fratrækkes inden investering) |
| `α` | Månedlig depotomkostningsprocent (fx 0,10 % = 0,001) |
| `r_t` | Realiseret månedligt investeringsafkast (stokastisk eller fast) |
| `U_t` | Udbetaling i måneden (`= 0` i opsparingsperioden) |
| `S` | Aftalt dækningssum (livsforsikringens forsikringssum) |
| `μ_x` | Bedste skøn for dødelighedsintensitet ved alder `x` pr. år |
| `μ_x⁺` | Konservativ dødelighedsintensitet: `μ_x⁺ = λ · μ_x` |
| `λ` | Sikkerhedsfaktor, typisk 1,05–1,20 |
| `x` | Forsikredes alder (opdateres løbende: `x = x₀ + t/12`) |
| `μ` | Forventet log-afkast pr. år (GBM-parameter) |
| `σ` | Volatilitet pr. år (GBM-parameter) |
| `r_f` | Risikofri rente pr. år (bruges under risikoneutrale mål) |
| `ε_t` | Standardnormalt stød, `ε_t ~ N(0,1)` |

---

## Kerneformler

### 1. Livsforsikringspræmie (månedlig)

```
δ_liv,t = S · μ_x⁺ · (1/12)
```

Dødelighedsintensiteten modelleres med **Gompertz-Makeham**:
```
μ_x = A + B · c^x
```
Standardparametre: `A = 0.0007`, `B = 0.00005`, `c = 1.095`

Konservativ kalibrering: `μ_x⁺ = λ · μ_x`, typisk `λ ∈ [1.05, 1.20]`

### 2. Månedlig depotfremregning (obligatorisk rækkefølge)

```
Trin 1 — Nettobidrag:
  D_t* = D_t + π - δ_liv,t

Trin 2 — Investeringsafkast (GBM, månedlig):
  r_t = exp( (μ - σ²/2) · (1/12)  +  σ · √(1/12) · ε_t ) - 1
  [deterministisk tilstand: sæt ε_t = 0]

Trin 3 — Depotopdatering:
  D_{t+1} = D_t* · (1 + r_t) · (1 - α) - U_t

Trin 4 — PAL-skat (bogholderi — påvirker IKKE depotet):
  PAL_t = (0.153 / 12) · max(D_t* · r_t, 0)
  PAL_aconto += PAL_t
```

---

## Vigtige invarianter — må aldrig brydes

1. **PAL fratrækkes IKKE depotet.** Den opgøres udelukkende som en acontoforpligtelse på selskabsniveau og nulstilles ved faktisk afregning til SKAT.
2. **Livsforsikringspræmie fratrækkes i Trin 1** (inden investering), ikke i Trin 3.
3. **Trinrækkefølgen 1 → 2 → 3 → 4 er obligatorisk** og må ikke ændres.
4. **`D_t ≥ 0` til enhver tid** — depotet kan ikke blive negativt.
5. **Alderen opdateres løbende:** `x = x₀ + t/12` — `δ_liv,t` stiger derfor med alderen.
6. **Under livrente-konvertering** (ved pensionsalder) omdannes depot til livsvarig månedlig ydelse via aktuarmæssige konverteringsfaktorer (forventet restlevetid + renteniveau).
7. **Negativ afkast-måned:** `PAL_t = 0` og akkumuleret underskud fremføres til reduktion af fremtidige PAL-betalinger.

---

## Interaktive elementer (HTML-implementering)

Følgende kontroller skal indgå i HTML-filen:

| Parameter | Type | Standardværdi |
|---|---|---|
| Startdepot `D_0` | Talindtastning / slider | 500.000 kr. |
| Månedlig præmie `π` | Talindtastning / slider | 3.000 kr. |
| Nuværende alder `x₀` | Talindtastning | 40 år |
| Pensionsalder | Talindtastning | 67 år |
| Produkt | Radio-knapper | Ratepension |
| Livsforsikring til/fra | Checkbox | Til |
| Dækningssum `S` | Talindtastning | 500.000 kr. |
| Sikkerhedsfaktor `λ` | Slider | 1,10 |
| Forventet afkast `μ` | Slider | 6 % p.a. |
| Volatilitet `σ` | Slider | 15 % p.a. |
| Depotomkostning `α` | Slider | 0,10 % / md. |
| Udbetalingsperiode (Rate) | Talindtastning | 15 år |
| Monte Carlo-stier | Talindtastning | 200 |

**Grafer:**
- **Graf 1:** Deterministisk depotudvikling (ε_t = 0) fra nu til udbetalingsperiodens afslutning
- **Graf 2:** Monte Carlo-bånd — percentiler P10, P50, P90 over `n` simulerede stier
- **Nøgletal-panel:** Forventet depot ved pensionsalder, estimeret månedlig ydelse, akkumuleret PAL-forpligtelse

Alle grafer og nøgletal opdateres **live** ved enhver parameterændring.

---

## HTML-filstruktur

```
pension.html
├── <head>
│   ├── <script src="Chart.js CDN">
│   └── <style> (inline CSS, responsivt layout)
└── <body>
    ├── <section id="controls">     ← alle sliders og inputs
    ├── <canvas id="chart-depot">   ← deterministisk depotgraf
    ├── <canvas id="chart-mc">      ← Monte Carlo-bånd
    ├── <div id="summary">          ← nøgletal
    └── <script>
        ├── // Beregningsfunktioner
        │   ├── beregnLivsforsikringsPraemie(S, mu_x_plus)
        │   ├── beregnGBM(mu, sigma, epsilon)
        │   ├── fremregn(params, stokastisk=false)   ← kerne-loop
        │   └── beregnPAL(D_star, r_t)
        ├── // Monte Carlo
        │   └── monterCarlo(params, n=200)
        ├── // Visualisering
        │   ├── initCharts()
        │   └── opdaterCharts(data)
        └── // Event listeners
            └── controls.addEventListener('input', opdater)
```

---

## Verificeringseksempel (bruges til at validere implementeringen)

Givet: `D₀ = 500.000 kr.`, `π = 3.000 kr.`, alder = 40, `S = 500.000 kr.`,
`μ₄₀⁺ = 0,00180 pr. år`, `α = 0,10 % / md.`, `r₀ = 0,80 %` (fast, ε = 0).

| Trin | Beregning | Resultat |
|---|---|---|
| Livsforsikringspræmie | 500.000 × 0,00180 / 12 | 75 kr. |
| Nettobidrag | 500.000 + 3.000 − 75 | 502.925 kr. |
| Efter afkast (0,80 %) | 502.925 × 1,0080 | 506.948 kr. |
| Depotomkostning (0,10 %) | 506.948 × (1 − 0,0010) | **506.441 kr.** |
| PAL (bogholderi) | (0,153 / 12) × (502.925 × 0,008) | 51 kr. |

Implementeringen er korrekt, når `D₁ = 506.441 kr.` og `PAL₁ = 51 kr.` for disse inputværdier.

---

## Produktforskelle ved udbetalingsperioden

| | Ratepension | Livrente | Aldersopsparing |
|---|---|---|---|
| Udbetalingsform | Fast månedlig rate i aftalt periode (min. 10 år) | Livsvarig ydelse (konverteres ved pensionsalder) | Rate eller engangsudbetaling |
| `U_t` beregning | `Depot / resterende_måneder` | Fastsat ved konvertering via aktuarfaktor | Aftalt plan |
| Overlevelsesgevinster | Nej | Ja — indgår i konverteringsfaktoren | Nej |
| Skattefri udbetaling | Nej | Nej | **Ja** |
| Depot ved død | Udbetales til begunstigede | Ophører (medmindre ægtefælledækning) | Udbetales til begunstigede |
