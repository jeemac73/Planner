# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file retirement planning web application (`Index.html`). There is no build system, no package manager, no bundler, and no test suite — the entire app (HTML, CSS, JavaScript) lives in one file. Open it directly in a browser.

## Architecture

### Single-File Structure

Everything is in `Index.html`:
1. `<head>` — Google Fonts import (DM Sans, DM Mono) and all CSS in a `<style>` block
2. `<body>` — HTML structure: API modal, sticky header, horizontal tab bar, tab content divs
3. `<script>` — All JavaScript at the bottom of the body

### State Objects

There are five independent state objects, each persisted to `localStorage`:

| Variable | localStorage key | Purpose |
|---|---|---|
| `P` | `rp_profile` | Main profile (age, savings, income, allocations) |
| `HC` | `rp_hc` | Healthcare bridge inputs |
| `RC` | `rp_rc` | Roth conversion inputs |
| `WD` | `rp_wd` | Withdrawal sequencing inputs |
| `budgetCategories` | `rp_budget` | Budget category array |
| *(array)* | `rp_holdings` | Investment holdings |
| *(string)* | `rp_api_key` | Anthropic API key |

All state reads use `safeLocalGet()` / `safeLocalSet()` wrappers that silently swallow `localStorage` errors (e.g., Safari private mode).

### Render Loop

`renderAll()` is the single re-render entry point. It is called:
- On `DOMContentLoaded` after `initFields()` runs
- On every `updateField()` call (Runway/Tax form inputs)
- On every tab switch via `switchTab(id)`

`renderAll()` only renders the currently active tab — there is a guard `if (currentTab==='X') renderX()` for each tab. This means tabs like Healthcare, Roth, and Withdrawal have their own dedicated `renderHealthcare()` / `renderRoth()` / `renderWithdrawal()` functions that also call their own `readXFields()` before computing.

`getCalcs()` computes all derived values from `P` (projected nest egg, required nest egg, tax) and is called at the start of every render cycle.

### Holdings vs. Profile Sync

Investment holdings are stored independently in `rp_holdings`. When a holding is added/edited/deleted, `syncHoldingsToProfile(holdings)` is called to derive and overwrite the account buckets in `P` (`rothBalance`, `traditionalIRABalance`, `taxableAccountBalance`, `otherSavings`).

### Dual `expectedReturn` Input

The Runway tab and Investments tab each render a separate `<input>` for expected return (`f-expectedReturn` and `f-expectedReturn2`). `updateField('expectedReturn', ...)` syncs whichever field is not currently focused.

### AI Advisor Tab

The AI Advisor makes a direct `fetch` to `https://api.anthropic.com/v1/messages` from the browser using the user-supplied API key. It uses `claude-sonnet-4-20250514` with a context-rich system prompt built from the current `P`, `HC`, and holdings state. The `anthropic-dangerous-direct-browser-access: 'true'` header is required for browser-side API calls.

### Washington State Assumptions

The Healthcare tab is built with WA-specific constants (`WA_STATE` object):
- No state income tax (Roth withdrawals don't affect MAGI or ACA subsidy)
- WA Apple Health (Medicaid) threshold: 138% FPL
- WA Healthplanfinder benchmark silver premiums (hardcoded ~$8,400/yr at 55, ~$9,600/yr at 60+)
- ACA subsidy calculations use 2024 enhanced APTC rules (no income cliff above 400% FPL)

Tax brackets throughout use **2024 IRS values** (both the `calcTax()` helper for the overview and the `getBrackets()` / `calcTaxOn()` functions for Roth conversion).

## Design System

CSS custom properties defined in `:root`:
- `--bg` / `--bg2` — dark navy backgrounds
- `--blue` / `--green` / `--amber` / `--red` / `--purple` — semantic accent colors
- `--text` / `--muted` / `--dim` — text hierarchy
- `--border` / `--card` — subtle white-alpha borders/fills

Component CSS classes: `.card`, `.stat-card`, `.stat-grid`, `.field-group`, `.field-input`, `.tab-btn`, `.insight`, `.f-table`, `.glide-row`, `.alloc-bar`, `.bar-chart`, `.msg-bubble`.

Number display uses `fmt()` (USD currency, no decimals) and `fmtPct()` throughout. Monospace font (`DM Mono`) is applied to all numeric outputs.

Mobile-first responsive: single-column below 600px, multi-column above. Uses `safe-area-inset-*` env vars for iOS notch/home-bar padding.
