# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Standalone solar system configurator for **Winkel Photovoltaik GmbH** — a German PV company. Single-page, client-side HTML application with no build tools, no frameworks, no external dependencies.

## Running the App

Open the HTML file directly in a browser — no server or build step required:

```bash
open "Interaktive Preisliste Winkel Photovoltaik GmbH BETA.html"
# or with live reload:
python3 -m http.server 8080
```

## Architecture

Everything lives in **`Interaktive Preisliste Winkel Photovoltaik GmbH BETA.html`** (~2,600 lines), structured as clearly-commented sections:

1. **CSS** — all styles inline in `<style>`, dark theme with CSS variables in `:root`
2. **HTML** — static skeleton with 7 step panels (`step-panel` class), header with live price, footer with nav buttons
3. **DATA** — embedded product catalog: `SYSTEMS` (module counts/prices), `INVERTERS`, `BATTERIES`, `ADDONS`, pricing constants
4. **STATE** — single global `state` object + `state.wkb` sub-object for all financial/consumption parameters
5. **HELPERS** — `fmtN()` (integer formatting), `fmtD()` (decimal formatting), `getKwp()`, `getTotalPrice()`, `getAnnualProd()`, `getSelfSuff()`, `getAnnualSavings()`, `getPayback()`, `get20YearSavings()`
6. **RENDER** — `renderStep0()` through `renderStep6()`, `renderStepNav()`, `renderLivePrice()`, `buildWkbChart()` — full re-render on every state change
7. **ACTIONS** — `nextStep()`, `prevStep()`, `goToStep()`, `restartApp()`, `setWkb()` with 150ms debounce
8. **INIT** — bootstraps on `DOMContentLoaded`

### 7-Step Wizard Flow

`Step 0 (Startseite/Consumption)` → `Step 1 (Anlagengröße)` → `Step 2 (Wechselrichter)` → `Step 3 (Speicher)` → `Step 4 (Extras)` → `Step 5 (Übersicht)` → `Step 6 (WKB)`

### State Management

All user selections live in one global object. `state.wkb` holds all financial parameters (strompreis, preiserhoehung, consumption breakdown, autarkiegrad, eigenverbrauchsanteil, etc.) shared between Step 0 and the WKB (Step 6). Changes via `setWkb()` trigger debounced re-renders and keep `state.consumption` in sync.

### Calculation Engine

Two parallel calculation paths that MUST stay consistent:
- **Helper functions** (`getAnnualSavings()`, `get20YearSavings()`, `getPayback()`) — used in Step 5 summary
- **`calcWKB()`** — detailed 20-year loop used in Step 6

Both MUST use `state.wkb.strompreis`, `state.wkb.preiserhoehung`, and `state.wkb.einspeiseverguetung`. Never use the legacy constants `GRID_PRICE` or `ANNUAL_INCREASE`.

Key parameters:
- Solar yield: `SOLAR_YIELD = 1000` kWh/kWp/year (1 kWp ≈ 1.000 kWh)
- Self-sufficiency: `0.30 + batteryKwh * 0.04`, capped at 85%
- EEG feed-in tariff: fixed for 20 years only (not 25)
- Financing: standard annuity formula in `calcFinancing()`
- 25-year comparison (Kapitalkonto) in renderStep6 includes investment + Reststrom − EEG

### Dual Display Mode (Cash vs. Financing)

When `state.wkb.finActive` is true, prices throughout the UI show €/Monat with Gesamt below. When false, full prices in €. This affects Steps 2 (inverters), 3 (batteries), 4 (add-ons), and the WKB.

### Key Data Files

- **`PL Winkel Photovoltaik GmbH Preisliste - Stand 16.03.2026.csv`** — authoritative pricing reference. When pricing changes, update both the CSV and the matching constants in the HTML.

## Localization

All UI text is **German**. Numbers use German formatting (`,` as decimal, `.` as thousands separator) via `fmtN()` and `fmtD()`. Maintain this convention everywhere.

## Important Conventions

- All inputs use `oninput` (not `onchange`) with debounce via `setWkb()` to prevent focus loss during re-render
- Stromkosten labels must include "(brutto)"
- Strompreis displays with 2 decimal places (0,42), Strompreissteigerung with 2 decimal places (6,00%), Einspeisevergütung with 3 decimal places (0,078)
- Footer always shows: "© Winkel Photovoltaik GmbH — Alle Rechte vorbehalten."
- Battery subsidy (1.500 €) is active via `SUBSIDY_ACTIVE` flag
