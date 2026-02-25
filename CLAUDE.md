# CLAUDE.md – Instruktioner til Claude Code

## Projektbeskrivelse

Du bygger en dansk, interaktiv pension-visualiseringshjemmeside. Målgruppen er almindelige danskere der gerne vil forstå deres pensionsbeslutninger. Tonen er venlig, pædagogisk og ikke-finansiel-jargon.

Hjemmesiden er **ikke** et rådgivningsværktøj. Den viser konsekvenser af beslutninger, ikke anbefalinger.

## Byggerækkefølge (følg denne rækkefølge!)

### Fase 1 – Grundstruktur ✅ TODO
- [ ] Opsæt Vite + React + Tailwind + Recharts
- [ ] Lav `App.jsx` med navigation mellem de tre sektioner
- [ ] Implementer dansk tekst fra `src/content/text.da.js`
- [ ] Lav simpelt, rent layout (hvid baggrund, sans-serif, god læsbarhed)

### Fase 2 – Sektion 1: Indbetaling og pensionsalder
- [ ] Inputfelter: alder, løn, nuværende opsparing, indbetalingsprocent, pensionsalder
- [ ] Beregning: opsparing ved pension (risikofri rente, ingen afkast-usikkerhed)
- [ ] Graf: opsparing over tid som kurve
- [ ] Formel vist tydeligt: `S(t) = S_0 * (1+r)^t + P * Σ(1+r)^i`

### Fase 3 – Sektion 2: Produktsammensætning
- [ ] Slider: fordeling mellem alderspension / ratepension / livrente (sum = 100%)
- [ ] For ratepension: vælg udbetalingsperiode (5, 10, 15, 20 år eller til død)
- [ ] For livrente: brug Gompertz-Makeham til at beregne forventet ydelse
- [ ] Vis månedlig ydelse for hvert produkt side om side
- [ ] Vis "overlevelsesgevinst" – hvad tjener du på at overleve?
- [ ] Graf: kumulativ udbetaling over alder (ratepension stopper, livrente fortsætter)

### Fase 4 – Sektion 3: Investeringsrisiko
- [ ] Slider: risikoniveau (lav/mellem/høj – mappes til σ og μ i GBM)
- [ ] Monte Carlo simulation (1000 paths) af opsparing frem til pension
- [ ] Vis percentiler (10%, 50%, 90%) som fan-chart
- [ ] Sammenlign med risikofri rente fra Sektion 1

### Fase 5 – Forfining
- [ ] Mobilvenligt layout
- [ ] Smooth scroll mellem sektioner
- [ ] Tilføj forklaringsbokse til fagtermer (tooltip eller expandable)
- [ ] Deploys til GitHub Pages

## Aktuarielle beregninger (src/utils/actuarial.js)

### Gompertz-Makeham mortalitet
```js
// Hazard rate: μ(x) = A + B * c^x
// Typiske danske parametre (ca.):
const A = 0.0007;
const B = 0.00005;
const c = 1.095;

function hazard(x) { return A + B * Math.pow(c, x); }

// Overlevelsessandsynlighed fra alder x til x+t
function survival(x, t, steps = 1000) {
  const dt = t / steps;
  let s = 1;
  for (let i = 0; i < steps; i++) {
    s *= (1 - hazard(x + i * dt) * dt);
  }
  return s;
}

// Nutidsværdi af livrente fra pensionsalder x, med rente r
function annuityPV(x, r, maxAge = 120) {
  let pv = 0;
  for (let t = 0; t <= maxAge - x; t++) {
    pv += survival(x, t) * Math.pow(1 + r, -t);
  }
  return pv;
}

// Månedlig ydelse fra opsparing S ved pensionsalder x
function annuityPayment(S, x, r) {
  return S / (annuityPV(x, r) * 12);
}
```

### Ratepension
```js
function ratePayment(S, years, r) {
  // Annuitet: S / Σ(1+r)^(-t) for t=1..years*12
  const monthly_r = Math.pow(1 + r, 1/12) - 1;
  const n = years * 12;
  const pv_factor = (1 - Math.pow(1 + monthly_r, -n)) / monthly_r;
  return S / pv_factor;
}
```

## Design-principper

- **Simpelt og rent** – ingen unødige elementer
- **Dansk** – al tekst på dansk, brug komma som decimalseparator i visning
- **Pædagogisk** – vis formler, men forklar dem i plain dansk
- **Interaktivt** – grafer opdateres live ved input-ændringer
- **Ikke-sælgende** – ingen produktanbefalinger, kun konsekvenser

## Farvepalette (forslag)
- Primær: `#1e3a5f` (mørk blå)
- Accent: `#e8a020` (varm orange)  
- Baggrund: `#f8f9fa`
- Tekst: `#212529`
- Graf-farver: blå, orange, grøn for hhv. alderspension, ratepension, livrente

## Vigtige noter

- Alle beregninger er **illustrative** – ikke finansiel rådgivning
- Tilføj disclaimer øverst på siden
- Brug dansk skat kun hvis det tilføjer pædagogisk værdi (det komplicerer beregningerne)
- Start **altid** med Fase 1 færdig før du går til Fase 2
