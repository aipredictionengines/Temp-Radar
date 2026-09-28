# Temp Radar

**Polymarket daily-high temperature radar — part of AI PREDICTION ENGINE by Digital Drago.**

Temp Radar is a single, self-contained HTML file that watches every open Polymarket *"Highest temperature in … on …"* market for **today, tomorrow and markets that are settling**, compares live station observations and seven weather models against the order book, and flags buckets where the model disagrees with the price. An optional Gemini layer (your own Google AI Studio key) adds a written analysis per city and a ranked scan of the whole board.

> **Signal-only.** Temp Radar never places orders and never touches a wallet. It tells you where it sees an edge; you decide and execute manually.

---

## Features

- **Auto-discovers markets.** Pulls all open daily-high markets from Polymarket (≈50 cities per day) and reads the resolution station, unit (°C / °F) and bucket ladder from each market's own rules.
- **Local-day aware.** Every market is classified by the station's local calendar day:
  - **Today** – the day is in progress at the station.
  - **Tomorrow** – forecast-only markets.
  - **Settling** – the local day is over but the market is still open.
- **Live observations.** Hourly and special METAR reports for each resolution station: current temperature, 2-hour trend and max-so-far.
- **Seven weather models.** ECMWF IFS, GFS, ICON, GEM, Météo-France, UKMO and JMA hourly forecasts, **bias-corrected** against the station's own recent observations.
- **Bucket probabilities.** A Monte Carlo model turns the corrected forecasts into a probability for every bucket, respecting whole-degree rounding and the max already observed.
- **Edge vs. order book.** Live bid/ask from the Polymarket CLOB, net of the estimated taker fee, gives a YES or NO edge for each bucket. Anything above your threshold is flagged as a signal.
- **City detail view.** Temperature curve (observed vs. projection vs. model range), the full bucket ladder, per-model table and the latest METAR / TAF.
- **Gemini analysis (optional).**
  - Per-city structured analysis: probabilities, pick, confidence, drivers, risks, and an answer to your own question ("I hold 20°C YES at 0.28 — hold or hedge?").
  - Board-wide **AI scan** that ranks the best opportunities, with optional scheduled scans.
- **Runs with no server.** Everything fetches directly from the browser, and there is no build step.

---

## Quick start

1. Download `temp-radar.html` and open it in Chrome (or any modern browser).
2. Wait a few seconds while markets, observations, models and prices load. The status pills in the header turn green as each source loads.
3. *(Optional)* Enable Gemini:
   1. Open **Settings**.
   2. Paste your **Google AI Studio API key**. Create one at [aistudio.google.com](https://aistudio.google.com/).
   3. Click **Load models**. The newest *Flash* model is pre-selected.
   4. Click **Save**.
4. Click any row to open the city detail. Click **AI** on a row, or **AI scan** in the toolbar, to run Gemini.

### Host it (Cloudflare Pages)

1. Rename `temp-radar.html` to `index.html`.
2. In Cloudflare, go to **Workers & Pages → Create → Pages → Upload assets** and upload the folder.

Your API key is never part of the page. Each visitor enters their own key in Settings.

---

## Settings

| Setting | Default | What it does |
|---|---|---|
| Google AI Studio API key | — | Used only for calls to `generativelanguage.googleapis.com`, sent from your browser. |
| Remember on this device | off | Stores the key in this browser's local storage. Off = the key is forgotten when the tab closes. |
| Gemini model | newest Flash | Any model your key can call with `generateContent`. |
| AI answer language | English | English or Bulgarian. |
| Auto AI scan | off | Re-runs the board scan every 15 / 30 / 60 min. Each run costs tokens. |
| Google Search grounding | off | Lets Gemini search the web. When on, the JSON schema is not enforced, so parsing is best-effort. |
| Signal threshold | 6¢ | Minimum net edge (after the estimated fee) for a bucket to be flagged. |
| Subtract est. taker fee | on | Applies the market's fee schedule to the edge. |
| CORS proxy | — | Optional Cloudflare Worker URL that enables TAF and the freshest METAR (see below). |

---

## How the model works

1. **Observations.** Every METAR for the station is converted to the market's unit and rounded to whole degrees, as the resolution source does. *Max so far* is the highest rounded reading in the local calendar day.
2. **Bias correction.** For each model: `bias = observed − model` over the last 6 hours.
   - Recent hours count more (weight `1 / (1 + hours ago)`).
   - The bias is capped at ±4 °C (±7 °F).
   - It is applied to the remaining hours of the day and fades with lead time (`× e^(−lead/10h)`).
3. **Distribution.** 4,000 Monte Carlo samples cycle through the models.
   - Each sample is the model's adjusted remaining-day max plus Gaussian noise. The noise σ is `min(1.6, 0.45 + 0.06 × lead hours)` °C, ×1.8 for °F.
   - The result is rounded to whole degrees.
   - For today it is floored at the observed max.
   - The sample counts per bucket become the probabilities.
4. **Edges.**
   - `Edge YES = p − ask − fee(ask)`
   - `Edge NO  = bid − p − fee(1 − bid)`
   - `fee(x) ≈ rate × (x · (1 − x))^exponent`, using the market's own `feeSchedule` (weather markets currently use rate 0.05, exponent 1).
5. **Status tags.**
   - **peak passed** – the observed max already covers more than 90% of the probability and the local time is after 14:00.
   - **day done** – no forecast hours remain in today's local day.
   - **final** – the market is in the Settling tab and resolved by observations.

The Monte Carlo is seeded per market and refreshed every 10 minutes, so probabilities don't flicker between refreshes.

---

## Data sources

| Data | Source | Key | Notes |
|---|---|---|---|
| Markets & rules | Polymarket Gamma API | none | Refreshed every 15 min. |
| Bid / ask | Polymarket CLOB `/prices` | none | Refreshed every 60 s. Falls back to Gamma prices if unavailable. |
| Station observations (METAR) | [Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/) (Iowa State University) | none | One batched request per refresh, every 3 min. Please keep usage modest. |
| Model forecasts | [Open-Meteo](https://open-meteo.com/) | none | Refreshed every 30 min. **The free API is for non-commercial use.** Commercial use requires an Open-Meteo API subscription. |
| Fallback coordinates | Open-Meteo Geocoding | none | Only for stations with no METAR feed. |
| AI analysis | Google Gemini API | **your key** | Usage and billing follow your Google AI Studio plan. |
| TAF + freshest METAR | aviationweather.gov (NOAA) | none | **Only through the optional proxy.** aviationweather.gov blocks browser (CORS) requests. |

**Resolution sources.** Most markets resolve on the NOAA time-series page (or Weather Underground) of an airport station, in whole degrees, for the local calendar day. Hong Kong resolves on the Hong Kong Observatory daily extract, at 0.1 °C precision. Temp Radar shows Hong Kong as **forecast-only, with no signals**. Always check the market's own rules on Polymarket before trading.

---

## Optional: TAF proxy (Cloudflare Worker)

`temp-radar-proxy.worker.js` is a ~40-line Worker that:

- forwards requests **only** to `aviationweather.gov`,
- adds CORS headers,
- caches each response for 60 seconds (their API asks for at most 1 request per minute per query).

**Deploy:**

1. In Cloudflare, go to **Workers & Pages → Create → Worker**.
2. Paste the file and click **Deploy**.
3. Copy the worker URL, e.g. `https://temp-radar-proxy.yourname.workers.dev`.
4. In Temp Radar, open **Settings → CORS proxy**, paste the URL and click **Save**.

When you select a city, the radar then loads its TAF (including `TX` max-temperature groups). It also merges the latest METAR straight from NOAA, and the TAF is passed to Gemini.

---

## Privacy

- There is no backend, no analytics and no tracking.
- The Gemini key goes only to Google's API, sent directly from your browser. It is stored only if you tick *Remember on this device*.
- Settings and the theme choice are kept in the browser's local storage. If storage is blocked, the radar still works; it just forgets settings.

---

## Limitations

- **Not financial advice.** Probabilities are model estimates. Weather can beat every model, and thin order books can show edges that disappear when you trade.
- **Observation delay.** The observation feed can lag a few minutes behind the station, and stations occasionally miss reports. Late-evening gaps are flagged in the Settling tab.
- **Fee estimate.** The fee is an estimate derived from each market's `feeSchedule`. Confirm actual costs on Polymarket.
- **API changes.** Public APIs can change without notice. If a status pill turns red, hover over it to see the error.

---

## Files

```
temp-radar.html              the radar (single self-contained file)
temp-radar-proxy.worker.js   optional Cloudflare Worker for TAF / fresh METAR
README.md
LICENSE                      MIT
```

---

## License

[MIT](LICENSE) © 2026 AI PREDICTION ENGINE By Digital Drago.

The license covers this code only. Market data, observations, forecasts and AI output remain subject to their providers' terms: Polymarket, Iowa Environmental Mesonet, Open-Meteo, NOAA / aviationweather.gov and Google.
