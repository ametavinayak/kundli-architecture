# Kundli Engine — architecture and verification notes

A deterministic Vedic astrology (Jyotish) computation engine serving a live commercial
site. The engine takes a date, time and place of birth and returns a complete chart:
planetary positions, houses, all sixteen divisional charts, dasha timelines, compatibility
scoring and strength calculations.

**The implementation is proprietary and not published here.** This document covers the
design decisions and — more usefully — how correctness was established, because that is
the part of the project worth reading.

---

## 1. The constraint that shaped the architecture

Every serious Jyotish implementation is built on the **Swiss Ephemeris** via `pyswisseph`.
It is excellent and it is the default choice.

It is also **AGPL-or-paid**. AGPL's network clause reaches a hosted API: serving chart
calculations over HTTP is conveyance, so shipping it commercially means either publishing
the entire surrounding application or buying a licence.

That constraint had two acceptable answers and one unacceptable one. The unacceptable one
was shipping AGPL code commercially and hoping nobody looked. The acceptable ones were
paying, or replacing the dependency.

**The dependency was replaced**, giving a production backend with zero licensing cost and
no copyleft obligations:

- **Skyfield** (MIT) for the numerical integration layer
- **JPL DE421** (public domain, NASA) as the ephemeris — covers births 1900–2050
- **Everything the ephemeris does not provide, implemented directly** from published
  astronomical formulae (Meeus / IAU): sidereal time, nutation, obliquity, Ascendant and
  Midheaven, Porphyry cusps, the mean lunar node (Rahu/Ketu), and the ayanamsa family

`pyswisseph` remains in `requirements-dev.txt` and appears nowhere in production. It has
exactly one job now: acting as the oracle that proves the replacement is correct.

This is the decision I would want to be asked about. It converted a licensing problem into
an astronomy problem, and the astronomy problem was tractable.

---

## 2. Establishing correctness

Replacing a reference implementation is only defensible if you can prove the replacement
agrees with it. Astrology software fails quietly — a wrong ayanamsa or an off-by-one
divisional mapping produces output that looks entirely plausible and is wrong in a way no
user will ever report.

So correctness is established against **independent oracles**, not against itself.

### Oracle 1 — Swiss Ephemeris parity

Planetary positions, nakshatra pada, retrograde state and combustion, checked across
**1901–2043**:

| Quantity | Agreement |
|---|---|
| Planets | ≤ 1 arcsecond |
| Moon | ≤ 3 arcseconds |
| Angles (Asc/MC) | ≤ 2 arcseconds |

### Oracle 2 — JHora cross-validation

A second, unrelated implementation, used to check the derived layers that Swiss Ephemeris
does not itself produce — divisional charts and dasha timelines.

### The combined run

100 randomised charts (seeded, reproducible), **17,893 individual checks, 17,893 passing**:

| Check | Count |
|---|---|
| Positions vs Swiss (< 5″) | 1,000 |
| Positions vs JHora (sign + 3′) | 900 |
| Nakshatra pada vs Swiss | 700 |
| Retrograde vs Swiss | 500 |
| Combustion vs Swiss | 600 |
| Dignity vs canonical tables | 206 |
| Divisional charts vs JHora (D2, D3, D7, D9, D10, D12, D16, D24, D27) | 6,300 |
| Vimshottari antardasha dates vs JHora (≤ 2 days) | 7,687 |

The antardasha check is the one I would point at. Dasha timelines are cumulative — an error
in the balance at birth propagates through every subsequent period, so 7,687 sub-period
boundaries agreeing to within two days across a century of charts is a much stronger
statement than any single position check.

### Golden anchors

Alongside the oracles, fixed reference values that must never move:

- Lahiri ayanamsa reference values
- The classical Pushya-Moon dasha example (Saturn 3y 1m 25d)
- BPHS ashtakavarga totals — a **337-bindu checksum**, which is self-validating: the
  sixteen divisional and eight-fold strength tables must sum to exactly 337, so a single
  misplaced bindu anywhere fails the sum
- Navamsa and trimsamsa boundary mappings, where off-by-one errors concentrate

37 test modules in total, including separate oracles for rise/set times and a
`verify_references` harness that emits its pass matrix as JSON rather than a boolean.

---

## 3. Layering

```
  engine/     pure-Python calculation core — no web dependencies at all
    astro/backends/    free_backend.py (production) | swiss_backend.py (oracle)
  api/        thin FastAPI JSON layer  (/api/chart, /api/match)
  web/static/ single-page UI, North and South Indian SVG chart styles
  data/ephe/  ephemeris data, auto-falls back to Moshier outside its range
  tests/      golden-value and invariant suite
```

The engine deliberately imports nothing from the web layer. Two consequences that were
the point of the split: the calculation core is testable without standing up a server,
and it is callable directly as a library — which is how an LLM-facing service later
reused it as a deterministic fact-checker, validating generated astrological claims
against computed truth instead of against another model.

Backends sit behind one interface, which is what makes the parity testing possible at all:
the same chart request can be routed through the free backend or the Swiss oracle and the
outputs compared field by field.

---

## 4. What the engine computes

- **D1 Rashi chart** — 9 grahas, Lagna, Whole-Sign houses plus Sripati Bhava Chalit
- **Shodashavarga** — all 16 divisional charts (D1–D60) per BPHS, including the unequal D30
- **Vimshottari dasha** — up to 4 nested levels with exact balance at birth
- **Panchang** — tithi, vara, nakshatra, yoga, karana
- **Ashtakoot Guna Milan** — 36-point matching with Nadi, Bhakoot and Mangal dosha
- **Ashtakavarga** — BAV and SAV
- **Shadbala (core)** — uchcha, kendradi, ojha-yugma, dig, paksha, naisargika
- **Yogas** — rules engine covering Mahapurusha, Gaja Kesari, Kaal Sarp, Raja and others

Output includes rendered SVG charts in English, Hindi and Bengali.

---

## 5. Limits

Stated because a design document that lists only strengths is marketing.

- **Shadbala is partial.** Saptavargaja, the kala sub-balas, chesta and drik bala are not
  implemented. What ships is the core six.
- **Ephemeris range is bounded.** DE421 covers 1900–2050; outside it the system falls back
  to Moshier, which is less accurate. Births outside that window are not equally trustworthy.
- **The free backend is verified against Swiss Ephemeris, not against reality.** Parity to
  one arcsecond means it reproduces the reference implementation, which is a different and
  weaker claim than astronomical truth. For this application it is the right target — users
  expect agreement with standard software — but it is worth being precise about.
- **Mean node only.** Rahu/Ketu use the mean lunar node, not the true node.

---

## 6. A correctness fix worth describing

Mangal (Manglik) dosha is the highest-stakes output in the system — it is the field that
makes people call off marriages — and the naive rule flags far more charts than the
classical texts intend, because it ignores cancellation.

The cancellation conditions were implemented: Mars in its own sign or exalted, and Jupiter
conjunct or aspecting Mars, both cancel the dosha. Severity was also made conservative,
reserving "high" for the genuinely severe placements (7th or 8th house, or a Lagna-and-Moon
double affliction) rather than applying it broadly.

Every outcome path — cancelled, partial, high, none — was verified against real charts
before the change shipped, and the change was propagated to the separate copy of the engine
running the LLM service so that the two could not silently disagree.

---

*Author: Vinayak Ameta. The engine runs in production behind a commercial site.*
