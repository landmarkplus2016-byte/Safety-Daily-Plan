# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A browser application for field operations daily site plan management (Vodafone Egypt / Landmark contractor). No build system, no server, no package manager — serve the project root over any static host or open `index.html` via a local server (required for `fetch('data/lists.xlsx')` to work).

## Running the App

Serve the project root with any static file server (e.g. VS Code Live Server, `npx serve .`, or Netlify). Opening `index.html` directly as a `file://` URL will fail the `fetch` for `data/lists.xlsx`.

CDN dependencies (loaded in `index.html` `<head>`):
- ExcelJS 4.4.0 (cdnjs) — Excel export
- SheetJS 0.18.5 (cdnjs) — parse `data/lists.xlsx` on startup
- Google Fonts (Syne, DM Sans)

## Git / Deployment

- Remote branch: `master` (default). Local branch is also `master`.
- GitHub Pages serves from `master` branch.
- After pushing, Pages takes ~1-2 min to redeploy. Always bump the SW cache timestamp on every deploy (see Service Worker section).

## File Structure

```
index.html        HTML shell — meta, CDN scripts, all markup
styles.css        All CSS — design system, components, overlays
app.js            All application JS + PWA SW registration
sites-data.js     SITES lookup table { siteId: [area, address, lat, lon] }
data/lists.xlsx   Single source of truth for all dropdown/datalist data
data/banner.jpg   Page banner image (replaces on GitHub → auto-refreshes via network-first SW)
service-worker.js PWA offline caching (versioned cache strategy)
manifest.json     PWA manifest
icon-192.png      PWA icon
icon-512.png      PWA icon
```

Script load order in `index.html` (order matters):
1. ExcelJS CDN
2. SheetJS CDN
3. `sites-data.js` — defines `SITES` global before app.js runs
4. `app.js` — consumes `SITES` and all DOM elements

## Architecture

### styles.css
Design system via `:root` CSS custom properties. Vodafone red `#E8192C` is the brand color. Grid utilities: `.grid-2`, `.col-span-2`. Component naming is BEM-like (`.site-block`, `.site-block-header`, `.section-card`).

### index.html — page structure (top to bottom)
- `<div id="lists-loading">` — full-screen spinner shown while `data/lists.xlsx` is fetched; replaced with error card on failure
- `<img class="page-banner">` — **fixed** banner at top of page; collapses on scroll; uses `height: auto` for natural aspect ratio (no cropping on any screen size)
- `<button class="fab-refresh">` — floating refresh button, bottom-right, above settings FAB
- `<button class="fab-settings">` — floating settings button, bottom-right corner (SVG icon, not emoji — renders consistently across platforms)
- `<template id="site-block-template">` — cloneable site entry unit (the core UI component)
- `<form id="planForm">` — **starts empty** (no site blocks on load); dynamically populated when user clicks "Add Sites"
- `#entries-section` — renders saved plans from localStorage; section header has a collapse/expand toggle arrow
- `#settingsPanel` — Settings modal (coordinator defaults)
- `#confirmModal` — shared confirmation dialog for destructive actions (delete entry, clear all, remove site)
- `#emailModal` — email composition modal
- `#toast` — floating notification
- `<datalist id="teamNames">` — engineer/supervisor name suggestions (populated from lists.xlsx)
- `<datalist id="contactNames">` — contact person name suggestions (populated from lists.xlsx)
- `<datalist id="driverNames">` — driver name suggestions (populated from lists.xlsx)

**No sticky/fixed header ribbon** — the red header bar was removed. Navigation is handled entirely by the two FABs.

### app.js — functional areas
- **Dynamic data loading:** `loadListsData()` — fetches `data/lists.xlsx` via SheetJS on startup, populates `TEAM`, `CONTACTS`, `CARS`, `LIST_SITE_TYPES`, `LIST_PROJECTS`, `LIST_SUB_CONTRACTORS`; calls `populateDatalists()` and `applyDropdownOptions(block)` on all existing blocks; hides loading overlay on success
- **Banner height tracking:** `_setBannerH()` — measures the banner's rendered `offsetHeight` and sets `--banner-h` CSS variable; called on image `load`, `resize`, and `requestAnimationFrame`. Container `padding-top` uses `calc(var(--banner-h, 180px) + 16px)` so content always clears the banner on any screen size.
- **Banner collapse:** scroll listener toggles `body.banner-collapsed` at `scrollY > 20`; CSS uses `transform: translateY(-100%)` to slide banner out and transitions `padding-top` on `.container` to 16px
- **Site block lifecycle:** `createSiteBlock()`, `addSite()`, `removeSite()`, `updateSiteUI()`; form starts with zero blocks — first block only appears when user clicks "Add Sites"; `removeSite()` allows removing the last block (returns to empty state); `updateSiteUI()` hides the save button when zero blocks are present
- **Dropdown helpers:** `applyDropdownOptions(block)` — fills siteType, projectName, subContractor selects from loaded lists; `populateDatalists()` — fills the three shared datalists
- **Settings:** `openSettings()` / `closeSettings()` / `saveCoordinator()` — manage default coordinator stored in localStorage; `applyDefaultContacts(block)` — pre-fills contactName/contactMob on new blocks from saved defaults
- **UI interactions:** `toggleBlock()` (accordion), `toggleRisk()` (risk checkboxes), `setToggle()` (permission toggle), `refreshSummary()` (collapsed state pills)
- **Confirmation dialog:** `showConfirm(message, okLabel, onOk)` — reusable modal for all destructive actions
- **Site ID autocomplete:** `showSiteAc()` / `hideSiteAc()` — case-insensitive prefix match against `SITES`; selecting triggers `lookupSiteId()` to auto-fill area, address, lat/lon
- **Auto-fill on input:** engineer/supervisor name → mobile (from `TEAM`), contact name → mobile (from `CONTACTS`), driver name → plate (from `CARS`)
- **Persistence:** `getEntries()` / `saveEntries()` using localStorage key `dailyPlanEntries_v2`
- **Edit flow:** `editPlan()` → `populateSiteBlock()` → form submit → `cancelEdit()`; cancel returns to empty form (no site blocks)
- **Use as New Plan:** `useAsNewPlan(planId)` — loads a saved plan's sites into the form as a fresh unsaved plan (does not enter edit mode); user can modify and save as new
- **Saved plans collapse:** `toggleAllPlans()` — toggles `plans-collapsed` class on `#entries-list` to hide/show all plan cards at once; arrow button sits next to "Saved Plans" heading
- **Export:** `downloadExcel()` — ExcelJS generates a 30-column `.xlsx` with color-coded section headers and status-based row colors (green=Update, red=Cancel)
- **Email:** `openEmailModal()` / `sendEmail()` — downloads Excel then opens a `mailto:` link
- **PWA:** Service worker registration at end of file

## UI Decisions

| Decision | Rationale |
|----------|-----------|
| Form starts empty (no site blocks) | Cleaner first-time experience; user explicitly adds sites |
| "Add Sites" button (not "Add Another Site") | Works as primary CTA whether 0 or N sites exist |
| Remove button always visible | Allows removing the last block to return to empty state |
| FAB for settings + refresh (bottom-right, stacked) | Thumb-friendly on mobile; keeps header area clean |
| No red ribbon header | More screen space; FABs handle all navigation |
| Banner `height: auto` + JS measurement | Prevents cropping at any screen width — fixed pixel heights always crop on some devices |
| `transform: translateY(-100%)` for collapse | Animates cleanly without layout shift; works with dynamic height |
| FAB uses SVG icon (not ⚙️ emoji) | Emoji renders blue on Android; SVG is always white |
| "Saved Plans" section-level collapse | Users want to hide all plans at once, not individually |
| "Use as New Plan" button (purple) | Distinct from Edit (orange) — makes clear it won't overwrite the original |

## Dynamic Data (data/lists.xlsx)

All dropdown and datalist values come exclusively from `data/lists.xlsx` (Sheet 1). Column layout:

| Col | Header | Populates |
|-----|--------|-----------|
| A | Site Engineer | `TEAM[].name` → `teamNames` datalist |
| B | Phone | `TEAM[].mob` (numeric → prepend "0" if 10 digits) |
| E | Contact Person | `CONTACTS[].name` → `contactNames` datalist |
| F | Phone | `CONTACTS[].mob` |
| H | Sub-Contractor | `LIST_SUB_CONTRACTORS` → subContractor select |
| I | Site Type | `LIST_SITE_TYPES` → siteType select |
| J | Project Name | `LIST_PROJECTS` → projectName select |
| L | Driver Name | `CARS[].name` → `driverNames` datalist |
| M | Car Plate No. | `CARS[].plate` |

Row 1 is the header row. Columns C, D, G, K are blank separators.

## Settings (localStorage)

| Key | Value shape | Effect |
|-----|-------------|--------|
| `defaultCoordinator` | `{ name, phone }` | Pre-fills contactName + contactMob on every new site block |
| `defaultEngineer` | `{ name, phone }` | Pre-fills supervisorName + supervisorMob on every new site block |

Saved via the ⚙️ Settings FAB; restored on open. Values are overwritten by `populateSiteBlock()` during edit mode.

## Data Model

Plans are stored in `localStorage['dailyPlanEntries_v2']` as JSON. Each plan has `_id` (timestamp), `_savedAt`, `_planDate`, and a `sites` array. Each site object has ~30 fields covering: site info, location (GPS + address), 5 risk categories (`riskWAH`, `riskElectrical`, `riskMechanical`, `riskManual`, `riskHotWork`), contact/team, transport, and permission/audit.

## Excel Export Structure

30 columns across 7 color-coded sections: Site Info (1–7, steel blue), Location (8–11, dark red), Risk Categories (12–17, dark green), Contact Person (18–20, dark purple), Engineers (21–24, gold), Transport (25–28, dark teal), Permission/Audit (29–30, dark orange). **Do not change this structure.**

## Hardcoded Configurations

- Contractor: `"Landmark"` (readonly input, never changes)
- Site Status options: Update, Cancel (hardcoded in template — not from lists.xlsx)

## Service Worker (service-worker.js)

Versioned cache strategy. Bump `CACHE_NAME` timestamp on every deploy:
```js
const CACHE_VERSION = 'v1';
const CACHE_NAME    = `daily-plan-${CACHE_VERSION}-20260325210000`;
//                                                  ↑ change this on each deploy
```
- `index.html` / bare `/` → **network-first** (always get latest on page load)
- `data/*` (lists.xlsx, banner.jpg, …) → **network-first** (always fetch fresh; fall back to cache if offline)
- All other same-origin assets → **cache-first + background revalidate**
- CDN assets → **network-first, fall back to cache**
- `clients.claim()` inside the `waitUntil` chain so new SW takes effect immediately

## Asset Embedding

`LMP Big Logo.jpg` is embedded as a base64 data URI directly in `index.html` (`class="page-logo"`) so the logo works on mobile without an extra HTTP request.

`data/banner.jpg` is **not** embedded — it is fetched at runtime via an `<img>` tag and served network-first by the service worker so replacing the file on GitHub is reflected immediately on the next page load.

## Key Files

| File | Purpose |
|---|---|
| `index.html` | HTML shell — markup only, no inline CSS or JS |
| `styles.css` | All application styles |
| `app.js` | All application logic + PWA SW registration |
| `sites-data.js` | SITES lookup table (site IDs → area, address, GPS) — auto-generated from `Site Address New.xlsx` |
| `data/lists.xlsx` | Live data source for all dropdowns and datalists |
| `data/banner.jpg` | Page banner image — replace to update; fetched network-first; displayed at natural aspect ratio |
| `service-worker.js` | PWA offline caching |
| `manifest.json` | PWA manifest |
| `LMP Big Logo.jpg` | Source logo (already embedded in index.html as base64) |
| `Site Address New.xlsx` | Source data for sites-data.js |
| `List.xlsx` | Root copy of data/lists.xlsx (source of truth for editing) |
