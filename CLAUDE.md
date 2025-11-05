# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **single-file HTML application** for calculating capital gains tax on stock sales. The entire application (HTML, CSS, and JavaScript) is contained in `index.html`. There are no build tools, package managers, or separate JavaScript/CSS files.

**Key Features:**
- Real-time capital gains and tax estimation
- Automatic tax rate calculation based on income, filing status, and NIIT (Net Investment Income Tax)
- Stock price integration via Alpha Vantage API with multi-key rotation and rate-limit handling
- Historical price lookup with sparkline visualization
- Transaction storage in browser localStorage
- Tax bracket headroom calculation (shows remaining room before jumping to next bracket)

## Development Workflow

### Running the Application
Simply open `index.html` in a web browser. No build step or local server is required for basic functionality.

For testing with live APIs or to avoid CORS issues:
```bash
python3 -m http.server 8000
# Then navigate to http://localhost:8000/index.html
```

### No Build, Lint, or Test Commands
This project has no build pipeline, linters, or test frameworks. All changes are made directly to `index.html`.

## Code Architecture

### Single-Page Application Structure
The `index.html` file contains three main sections (in order):
1. **Styles** (`<style>` tag ~lines 17-1440): CSS using Open Props design tokens
2. **HTML Structure** (~lines 1441-1605): Markup including custom `<pretty-slider>` web component
3. **JavaScript** (`<script>` tag ~lines 1605-2771): All application logic

### Core JavaScript Architecture

#### State Management
Global `state` object (~line 1791) contains:
- `shares`, `costBasis`, `taxRate`: Current calculation inputs
- `stockData`: Cached ticker data with sparklines and prices
- `transactions`: Array of saved transactions (persisted to localStorage)
- `apiOverrideMode`: Controls data source ('mock', 'cache', or 'live')

#### Tax Calculation System
Tax logic is centralized in `TAX_CONFIG_2025` (~line 1607):
- **Long-term brackets**: 0%, 15%, 20% based on income thresholds
- **Short-term brackets**: Ordinary income rates (10%-37%)
- **NIIT**: 3.8% additional tax on investment income above thresholds ($250k joint, $200k single)

Key functions:
- `getTaxRates()` (~line 2209): Returns applicable rates for given income/status
- `updateTaxLogic()` (~line 2251): Auto-calculates and applies correct tax rate
- `calculate()` (~line 2264): Core calculation engine, also auto-selects long-term 15% vs 20% bracket based on effective income (income + gains)

#### API Integration & Caching
Three-tier data strategy managed by `fetchSingleTickerData()` (~line 1928):

1. **Mock Data** (`getMockTimeSeries()` ~line 1844): Hardcoded realistic time series for TSLA/META/NVDA
2. **Cache** (localStorage key: `alphaVantageCache`): 2-hour expiry
3. **Live API** (`alphaVantageApiCall()` ~line 1889): Alpha Vantage TIME_SERIES_DAILY with multi-key rotation

API key rotation system:
- Multiple API keys in `apiKeys` array (~line 1790)
- `currentApiKeyIndex` tracks which key to try next
- Auto-rotates on rate-limit errors (`"API rate limit"` or `"premium"` in response)

#### Dev Panel
Hidden debug panel (toggle with `Cmd+D` or button) at ~line 63:
- Shows API calls, cache hits/misses in timeline view
- `logToDevPanel()` function (~line 1830) for logging
- Tax bracket editor for testing different years/rates
- API override controls (mock/cache/live)

#### Transaction System
Transactions are saved to localStorage:
- `saveTransaction()` (~line 2350): Adds transaction to state and localStorage
- `deleteTransaction()` (~line 2375): Removes transaction
- Stored in key `capitalGainsTransactions`
- Rolling totals calculated in `renderTable()` and `renderBackPanel()`

#### Card Flip UI
Front card shows calculator, back card shows saved transactions:
- `flipCard()` function manages transition
- Back panel includes tax bracket headroom display (shows $ remaining before next bracket)

## Important Implementation Notes

### Tax Rate Auto-Selection
The `calculate()` function (~line 2264) has special logic:
- For **long-term gains**: Auto-selects 15% or 20% bracket based on `effectiveIncome = income + totalGainLoss`
- For **short-term gains**: Always uses manually selected rate
- Override behavior controlled by `manualTaxRateOverride` flag

### Data Flow for Stock Prices
1. User selects ticker from dropdown → triggers `fetchStockData()`
2. `fetchStockData()` calls `fetchSingleTickerData()` for each ticker
3. `processTimeSeries()` (~line 2023) generates sparkline data points
4. `updateUIWithData()` (~line 2146) populates UI with latest price and sparkline

### LocalStorage Keys
- `alphaVantageCache`: Stock price time-series data (2hr expiry)
- `capitalGainsTransactions`: User's saved transactions array

### Custom Web Component
`<pretty-slider>` (~line 1444): Standalone web component for income input slider, defined with `customElements.define()`.

## Modifying Tax Brackets

To update tax brackets for a new year:
1. Locate `TAX_CONFIG_2025` (~line 1607)
2. Update `year`, `longTerm`, `shortTerm`, and `niit` threshold objects
3. Tax rates are decimal values (e.g., `0.15` = 15%)
4. Bracket values are income ceilings (e.g., `0.15: 600050` means 15% rate applies up to $600,050)
