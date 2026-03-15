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

### index.html
Key elements:
- `<div id="lists-loading">` — full-screen spinner shown while `data/lists.xlsx` is fetched; replaced with error card on failure
- `<header class="header">` — sticky red header with logo, date badge, and ⚙️ Settings button
- `<img class="page-banner">` — banner image loaded from `data/banner.jpg` (network-first; replace the file on GitHub to update immediately)
- `<template id="site-block-template">` — cloneable site entry unit (the core UI component)
- `<form id="planForm">` — dynamically populated with cloned template instances
- `#entries-section` — renders saved plans from localStorage
- `#settingsPanel` — Settings modal (coordinator + engineer defaults)
- `#confirmModal` — shared confirmation dialog for destructive actions (delete entry, clear all, remove site)
- `#emailModal` — email composition modal
- `#toast` — floating notification
- `<datalist id="teamNames">` — engineer/supervisor name suggestions (populated from lists.xlsx)
- `<datalist id="contactNames">` — contact person name suggestions (populated from lists.xlsx)
- `<datalist id="driverNames">` — driver name suggestions (populated from lists.xlsx)

### app.js — functional areas
- **Dynamic data loading:** `loadListsData()` — fetches `data/lists.xlsx` via SheetJS on startup, populates `TEAM`, `CONTACTS`, `CARS`, `LIST_SITE_TYPES`, `LIST_PROJECTS`, `LIST_SUB_CONTRACTORS`; calls `populateDatalists()` and `applyDropdownOptions(block)` on all existing blocks; hides loading overlay on success
- **Site block lifecycle:** `createSiteBlock()`, `addSite()`, `removeSite()`, `updateSiteUI()`; `createSiteBlock()` calls `applyDropdownOptions()` and `applyDefaultContacts()` on every new block; `removeSite()` shows a confirmation dialog before removing
- **Dropdown helpers:** `applyDropdownOptions(block)` — fills siteType, projectName, subContractor selects from loaded lists; `populateDatalists()` — fills the three shared datalists
- **Settings:** `openSettings()` / `closeSettings()` / `saveCoordinator()` / `saveEngineer()` — manage default coordinator and engineer stored in localStorage; `applyDefaultContacts(block)` — pre-fills contactName/contactMob and supervisorName/supervisorMob on new blocks from saved defaults
- **UI interactions:** `toggleBlock()` (accordion), `toggleRisk()` (risk checkboxes), `setToggle()` (permission toggle), `refreshSummary()` (collapsed state pills)
- **Confirmation dialog:** `showConfirm(message, okLabel, onOk)` — reusable modal for all destructive actions; wires `#confirmModal` buttons, cleans up listeners after use, closes on backdrop click
- **Site ID autocomplete:** `showSiteAc()` / `hideSiteAc()` — case-insensitive prefix match against `SITES`; selecting triggers `lookupSiteId()` to auto-fill area, address, lat/lon
- **Auto-fill on input:** engineer/supervisor name → mobile (from `TEAM`), contact name → mobile (from `CONTACTS`), driver name → plate (from `CARS`)
- **Persistence:** `getEntries()` / `saveEntries()` using localStorage key `dailyPlanEntries_v2`
- **Edit flow:** `editPlan()` → `populateSiteBlock()` → form submit → `cancelEdit()`
- **Export:** `downloadExcel()` — ExcelJS generates a 30-column `.xlsx` with color-coded section headers and status-based row colors (green=Update, red=Cancel)
- **Email:** `openEmailModal()` / `sendEmail()` — downloads Excel then opens a `mailto:` link
- **PWA:** Service worker registration at end of file

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

Saved via the ⚙️ Settings panel; restored on open. Values are overwritten by `populateSiteBlock()` during edit mode.

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
const CACHE_NAME    = `daily-plan-${CACHE_VERSION}-20260304120000`;
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
| `data/banner.jpg` | Page banner image — replace to update; fetched network-first |
| `service-worker.js` | PWA offline caching |
| `manifest.json` | PWA manifest |
| `LMP Big Logo.jpg` | Source logo (already embedded in index.html as base64) |
| `Site Address New.xlsx` | Source data for sites-data.js |
| `List.xlsx` | Root copy of data/lists.xlsx (source of truth for editing) |
