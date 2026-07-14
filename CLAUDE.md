# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Vedabite is a single-page marketing/product web app: a free Ayurvedic dosha quiz that funnels users into a paid personalized wellness plan. All content is in Spanish (`es-AR`, Argentine voseo — "sos", "comé", "tenés"). The live site is **vedabite.com**, served as static files via **GitHub Pages** (see `CNAME`). There is **no build step, no framework, no package manager, no tests** — `index.html` is the entire deployed application (~9,200 lines, ~1 MB), a single file with inline `<style>` and `<script>`.

## Deploy / develop

- **Deploy:** commit to `main` and push. GitHub Pages serves `index.html` at `vedabite.com`. Every commit in history is literally titled "index.html".
- **Preview locally:** open `index.html` in a browser, or `python -m http.server` then visit `localhost:8000`. No install needed. Note that Firebase, geolocation, and the photo-scanner API calls hit live production services even locally.
- **Cache busting:** the app self-manages cache via the `vb-version` meta tag + matching `VERSION` constant in the first inline script. When you ship a user-visible change, bump **both** (format `B1-<slug>-<YYYYMMDD>`). On version mismatch the app clears its localStorage caches and force-reloads with a `?v=` timestamp param.
- `vedabite_v3(5).html` is an **old standalone prototype** ("GPS del Bienestar"), not deployed and not linked from `index.html`. Ignore it unless explicitly asked; edits belong in `index.html`.

## Editing this file

- It's one giant file — use Grep to locate a function or `id="..."`, then Edit by anchor. Never try to read it top to bottom.
- Almost all JS is a single global-scope IIFE-free script; functions and data are plain globals (`Q`, `D`, `P`, `res`, `show()`, etc.). New helpers are typically added as `window.xxx` or top-level `function`.
- **Naming convention marks eras of the code:** short cryptic names (`show`, `scan`, `getRes`, `D`, `P`, `Q`) are the original quiz; the `vb*` / `_vb*` prefix (`vbAbrirPaso4`, `_vbCalcularPrakriti`, `window._vbDB`) marks the newer premium/plan/astrology layer. Match the prefix of whatever region you're editing.
- Copy-protection is intentional: the head script disables right-click, text selection, F12/devtools shortcuts, and stubs out `console.*` on load. Expect a silent console in production.

## Architecture / big picture

**Screen model.** The UI is a set of `.screen` divs (`s1`, `s1b`, `s1c`, `s2`, `s3`, `s4`); `show(id)` toggles the `active` class and manages the bottom `navBar`. Within the results screen, `showTab(name)` swaps `.tab-content` panels (`t-alimentos`, `t-recetas`, `t-astrologia`, `t-escaner`, …). The premium plan flow is a separate wizard of "Paso" screens driven by `vbAbrirPaso4`…`vbAbrirPaso9Rasayana` / `vbCerrarPasoN`.

**Quiz engine.** `Q` is the array of ~50 questions, each tagged with a dosha (`vata`/`pitta`/`kapha`). `renderQ()` shows one at a time; `answer()` accumulates into `ans={vata,pitta,kapha}`; `showResults()` computes `res` (dominant dosha `res.p` + percentages `res.pct`). This drives everything downstream.

**Content data tables** (large object literals near the top, ~line 2075+):
- `D` — per-dosha display content (traits, foods, recipes, schedules, exercises, mudras, chakras, breathing, meditation, health tips). This is the bulk of what the results tabs render.
- `P` — per-dosha food classifier (`g`=good, `b`=bad, `sub`=substitutions) used by the `scan()` menu analyzer.
- `VOCES` — three narrative "voice"/tone presets (`medico`, `ayurveda`, `amiga`) that reword the informe and astrology output.
- `VB_*` tables (`VB_RASHI_DOSHA`, `VB_NAKSHATRA_DOSHA`, `VB_GRAHAS_INFO`, `VB_BHAVAS`, `VB_PLAN_ALIMENTACION`, `VB_CONDICIONES_IMPACTO`, …) — Vedic astrology mappings and the premium plan generator inputs.

**Monetization / gating.** Free users get the quiz + basic results; the plan/scanner/astrology depth is gated. Region-aware checkout: `mpAR` (Mercado Pago subscription) for Argentina vs `lsIntl` (Lemon Squeezy) for everyone else, chosen by IP country from `ipapi.co`. Trial/paid state lives in localStorage and is read via `vbEsUsuarioPago()` / `vbDiasTrial()` / `aplicarGating()`. Checkout entry points: `goToPremiumCheckout()`, `vbCheckoutPaso7()`.

**Backends & third-party services** (all called directly from the browser):
- **Firebase Firestore** (`vedabite-1b49a`, compat SDK loaded via `gstatic` CDN) — lead capture and cross-device progress. `window._vbDB` is the handle; writes go through a `guardarLog`-style path that falls back to buffering leads in `localStorage` (`vb_leads_pendientes`) on failure. `window._vbHealth` tracks connectivity.
- **Photo scanner** — `https://analizarfotoayurveda-<...>.a.run.app` (a Google Cloud Run function) for AI Ayurvedic analysis of uploaded meal/face photos (`procesarFoto*`, `scan`-adjacent flows).
- **Astrology** — `json.freeastrologyapi.com` for planetary positions / natal chart (`vbApiPlanets`, `vbApiGeo`, `vbApiTimezone`, `calcAstro`).
- **Geo** — `ipapi.co` for country detection (drives checkout routing).
- **Analytics** — GA4 + Google Ads (`gtag`, IDs `G-HM7G4LZM70` / `AW-632212911`) and Microsoft Clarity (`w2xvsd3oxf`). Events are fired liberally throughout via `gtag('event',...)`, `window.clarity('event',...)`, and the internal `window._vbTrack` / `enviarEvento` helpers. When adding a user action, follow the existing pattern and emit tracking for it.

**Persistence.** State survives reloads through localStorage keys prefixed `vb_` (`vb_version`, `vb_pais`, `vb_admin`, `vb_leads_pendientes`, session/progress caches). `saveSession`/`loadSession`/`clearSession` manage resumable quiz sessions; `_vbSincronizarProgreso` / `_vbCargarProgresoDesdeFirebase` sync progress to Firestore when an email is known.

## Gotchas

- Secrets in this file (Firebase `apiKey`, analytics IDs, Clarity token) are **client-side public keys by design** — not a leak. Firestore is the security boundary; don't "fix" them.
- Because it's one file with global state, edits can have action-at-a-distance effects. After changing data tables or flow functions, click through the actual quiz → results → plan flow in a browser to confirm nothing downstream broke.
- Keep Spanish/voseo consistent with surrounding copy when editing user-facing strings.
