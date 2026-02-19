# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file, self-contained browser application for field operations daily site plan management (Vodafone Egypt / Landmark contractor). No build system, no server, no package manager — just open [daily-plan-app 20.html](daily-plan-app%2020.html) directly in a browser.

## Running the App

Open `daily-plan-app 20.html` directly in any modern browser. All dependencies load from CDN:
- ExcelJS 4.4.0 (from cdnjs) — Excel export
- Google Fonts (Syne, DM Sans)

## Architecture

The entire application lives in one file with three sections:

**CSS (lines ~9–285):** Design system via `:root` CSS custom properties. Vodafone red `#E8192C` is the brand color. Grid utilities: `.grid-2`, `.col-span-2`. Component naming is BEM-like (`.site-block`, `.site-block-header`, `.section-card`).

**HTML (lines ~287–551):** Key elements:
- `<template id="site-block-template">` — cloneable site entry unit (the core UI component)
- `<form id="planForm">` — dynamically populated with cloned template instances
- `#entries-section` — renders saved plans from localStorage
- `#emailModal` — email composition modal
- `#toast` — floating notification

**JavaScript (lines ~552–1075):** Functional areas:
- **Site block lifecycle:** `createSiteBlock()`, `addSite()`, `removeSite()`, `updateSiteUI()`
- **UI interactions:** `toggleBlock()` (accordion), `toggleRisk()` (risk checkboxes), `setToggle()` (permission toggle), `refreshSummary()` (collapsed state pills)
- **Persistence:** `getEntries()` / `saveEntries()` using localStorage key `dailyPlanEntries_v2`
- **Edit flow:** `editPlan()` → `populateSiteBlock()` → form submit → `cancelEdit()`
- **Export:** `downloadExcel()` — ExcelJS generates a 30-column `.xlsx` with color-coded section headers and status-based row colors (green=Update, red=Cancel)
- **Email:** `openEmailModal()` / `sendEmail()` — downloads Excel then opens a `mailto:` link

## Data Model

Plans are stored in `localStorage['dailyPlanEntries_v2']` as JSON. Each plan has `_id` (timestamp), `_savedAt`, `_planDate`, and a `sites` array. Each site object has ~30 fields covering: site info, location (GPS + address), 5 risk categories (`riskWAH`, `riskElectrical`, `riskMechanical`, `riskManual`, `riskHotWork`), contact/team, transport, and permission/audit.

## Excel Export Structure

30 columns across 7 color-coded sections: Site Info (1–7, steel blue), Location (8–11, dark red), Risk Categories (12–17, dark green), Contact Person (18–20, dark purple), Engineers (21–24, gold), Transport (25–28, dark teal), Permission/Audit (29–30, dark orange).

## Hardcoded Configurations

- Contractor: `"Landmark"` (readonly)
- Sub-contractors: In House Team, Connect, DAM Telecom, El-Khayal, New Plan, Upper Telecom
- Site types: GF, RT, PT/MP, COW, GRD, Indoor
- Projects: New site Rollout, Upgrade on existing, CM Civil, Fixed Account
