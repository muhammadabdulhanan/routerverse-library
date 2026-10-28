# RouterVerse Library

Central registry of per-model router control profiles. The app never
hardcodes model data: it loads profiles from this Library (local bundle
first, remote fetch later) through `LibraryRepository`.

Layout:

- `profiles/<profile_id>.json` — one profile per model, fields exactly as
  in section 4 of the plan (see `schema/router_profile.schema.json`).
- `index.json` — generated lookup index: `by_brand`, `by_oui`,
  `by_fingerprint`, plus profile metadata. Built by
  `scripts/build_index.py`; do not edit by hand.
- `vendor_oui.json` — MAC OUI prefix registry. Started from Hanan's RedMind
  brand data (`app_discover.py` OUI_VENDORS: 92 prefixes, 19 vendors),
  then merged with the full nmap `nmap-mac-prefixes` file (curated
  entries win on conflicts) and adjudicated against the IEEE MA-L + IAB
  CSVs over a filtered router-vendor subset. Current registry:
  39,248 24-bit prefixes, 6,262 28-bit MA-M prefixes, 6,893 36-bit MA-S
  prefixes, 19,562 vendors. Only the 24-bit `prefix_to_vendor` map fits
  the profile schema; MA-M/MA-S are preserved verbatim under separate
  keys, never truncated (a truncated MA-M/MA-S prefix would over-claim
  its whole parent OUI), and the app reads only `prefix_to_vendor`
  today, so they cannot cause false matches.
  Used by the Detection Engine for brand hints. Brand data only — no
  credential values live in this Library; profiles carry a
  `default_credentials_ref` string that points at public factory-default
  references, never the values themselves.
- `schema/router_profile.schema.json` — JSON Schema for a profile.
- `scripts/build_index.py` — stdlib-only validator + index builder.

Hash rule (implemented identically in Python here and in Dart in the app's
`ProfileVerifier`):

    profile_hash = sha256_hex(canonical_json(profile with profile_hash=""
                                             and signature=""))
    canonical_json: keys sorted, separators "," and ":", UTF-8, no ASCII
    escaping. The Ed25519 `signature` covers the same canonical bytes;
    both fields are blanked before hashing/signing, in Python and Dart
    alike.

Commands:

    python3 scripts/build_index.py                # validate + build index
    python3 scripts/build_index.py --write-hashes # fix hashes, then build

Trust levels: a profile is `tested` only after it works on a real router,
`community` when built from public documentation, otherwise `unverified`.
Registration-only entries (known model, driver pending) have an empty
`supported_controls` list and say so in `known_issues`.

## Sources and honesty (2026-10-28 base merge + sweep)

Profiles come from public sources only. No credential values anywhere;
no fingerprints or OUIs invented; nothing deleted for being weak or old
(weak/old rows are kept and labelled in `known_issues`, which is
backend-only data — the app UI never shows source labels, verified
flags, or provenance notes).

Counts after the sweep: 12,399 profiles total (5,088 base + 7,311
sweep-new). 120 profiles carry at least one HTTP fingerprint and can
actually match; 12,279 are markerless registrations that can never
match until real fingerprints exist. Driver types: web 10,525,
openwrt 1,871, api 3. Status: community 12,368, unverified 31.

- FingerprintHub (MIT), `web-fingerprint/` rules only — never `plugins/`
  (exploit POCs). Verified and unverified rules kept; the verified flag
  is preserved in `known_issues`. Favicon-only rules kept with a note
  that matching needs a future favicon fetch; bare generic markers kept
  with a "weak marker — low confidence match" label. Base merge: 80
  rules converted, 74 skipped with grouped reasons (NAS/storage,
  management software, non-router products, unattributable markers).
- rapid7/recog (BSD-2-clause): `http_servers.xml`, `html_title.xml`,
  `http_wwwauth.xml` (the cookies file has no cookie field in the
  schema and was not converted). Clean literal substrings only.
  Base merge: 68 entries converted.
- urbanadventurer/WhatWeb (GPL-2.0): facts only (titles/headers/realms),
  no Ruby code vendored. 4 facts applied (Netis, Arris, ZTE, Huawei
  Eudemon, Netcomm hotspot gateway).
- WikiDevi 2019 snapshot (unverified 2019 data, stale for post-2019
  models — verify before any driver work): embedded-system device
  table, brand/model/revision/OUI. Adapters and other non-embedded
  templates are not routers and were excluded. Base merge: 4,968 new
  profiles, 11 existing enriched.
- DeviWiki live (deviwiki.com, SMW ask API `w/api.php`, fetched
  2026-10-28): page title, FCC ID, Manuf, Manuf. mdl, OUI, OUI
  (ethernet), chips, year. Fetched in 8 chunks (wireless category
  5,690 pages, wired category 600 pages, plus Manuf comparator ranges
  <C, [C,G), [G,M), [M,S), [S,Z), >=Z totalling 11,009 pages),
  combined to 9,030 distinct titles after merging SMW subobject
  results into their base pages. Rows deduped against the WikiDevi
  dump FIRST on FCC ID, then on Manuf-model. When the display brand
  differs from the manufacturer, an ISP/brand variant row is created,
  marked "rebrand — manufacturer proven, admin surface unproven".
  Sweep results: 767 Manuf-model matches, 558 FCC matches (each also
  yielding a manufacturer variant row), 1,146 new manufacturer bases,
  1,815 new display variants, 652 enriched variants, 2,240 new
  title-fallback rows and 2,816 enriched fallbacks (category-proven
  pages without a manufacturer property; brand/model from the page
  title). 150 adapter/card titles filtered out; 1,353 Manuf-only rows
  without a usable join key skipped; 45 FCC-ambiguous rows not
  force-matched to any single model. NOTE: HTML Special:Ask paging
  was tried first and abandoned — offsets past ~5,500 rows silently
  repeat the first rows, and the ask API has the same cap past offset
  5000, which is why every query above stays under 10,000 results.
- OpenWrt 25.12.6 Firmware Selector `profiles.json` per target
  (90 of 94 subtargets have a profiles file; malta/* builds generic
  images with none): 2,150 titles, 3 skipped (empty vendor/model),
  1,889 unique (brand, model) → 1,520 new profiles, 366 existing
  enriched, `driver_type: "openwrt"`. No stock admin fingerprint is
  ever inferred from OpenWrt support — these profiles are markerless
  until a real stock fingerprint exists. Non-router-ish targets
  (Raspberry Pi, x86, SBCs) are included as dormant OpenWrt-supported
  devices; trim later if wanted.
- PTCL tender seed: PTCL Refurbished GPON ONTs RFI (18-Aug-2020,
  ptcl.com.pk), 13 rows under brand PTCL with the underlying ONT
  vendor/model noted; admin surface = ISP variant, unproven. No OUIs
  inherited from the underlying vendors (the tender gives none).
- Verified ISP mappings (Whirlpool AU evidence for Telstra rows):
  12 rows — Telstra Gateway / Smart Modem Gen 1 / Gen 2 / Gateway
  Frontier, Optus Sagemcom/Arcadyan models, Bell Home Hub 2000,
  Rogers Ignite XB7 and Hitron CODA-4582/4680. The hardware mapping is
  verified; the ISP admin surface is unproven (no fingerprint yet).
- Saudi ISP-context CPE: 7 rows, shipped ONLY with the flag
  "network-compatible, ISP issue unproven" — ZTE MC801A1, MU5001;
  Huawei H155-381/383, E5172s, B715; Nokia FastMile. No
  tender/support/manual source confirms ISP issue.
- TP-Link MiFi (M7200, M7350, M7650): vendor product pages on
  tp-link.com cited; admin fingerprint pending. (These three already
  existed from DeviWiki, so the sweep enriched them with the
  vendor-page note instead of creating duplicates.)
- OUI layer: nmap `nmap-mac-prefixes` (full file; 24-bit OUIs in
  `prefix_to_vendor`, 28-bit MA-M and 36-bit MA-S preserved verbatim
  under separate keys, never truncated). IEEE MA-L + IAB CSVs used as
  adjudicator over a filtered router-vendor subset only (joined on
  brand, never a profile source): 6,174 filtered rows → 5,849
  consistent, 4 conflicts kept (registry value kept, stability),
  7 unmapped vendors kept, 314 prefixes added, 19 nmap-only router
  prefixes kept. MA-M/MA-S CSVs returned 404 from IEEE's site, so
  MA-M/MA-S stay nmap-sourced.

NOT shipped, honestly: Etisalat/du rows from an Alibaba market-insight
page (candidates only, unverified — the one Etisalat profile in the
Library, Etisalat S3, comes from OpenWrt support, not from that page);
Ooredoo (Qatar), Batelco (Bahrain), Omantel (Oman), Kuwait ISPs as
brands (no verified list found — gaps, not backfilled from Saudi/UAE
lists); STC, Mobily, Zain as brands (zero profiles under those names;
Saudi coverage is vendor-branded rows with the ISP-context flag);
DD-WRT database (bot-check fetch barrier — needs a live-browser check
later, not a scrape); Xiaomi MiFi lists (no citable enumeration
found); Huawei E-series hotspots (E5577, E5785, E5573, E5330, E5372 —
zero in every source); FiberHome as a brand beyond the PTCL AN5506
row (1 profile); specific models HG6245D, AN5506, H168N, H108N,
HG6245, H267N, F660, F670L as standalone fingerprints (absent from
WikiDevi; PTCL rows cover F660/F670L/AN5506 as ISP variants only).
Gulf/ISP coverage therefore rests on vendor-family profiles (Technicolor,
Sagemcom, Arris, Nokia, Huawei, ZTE fallbacks + models) matching stock
vendor UIs; ISP-customized UIs with ISP branding in title/realm are
unmatched.

Effective coverage, stated plainly: the Library holds 12,399 profiles,
but only the 120 match-capable ones can detect anything today; the rest
are dormant registrations preserved for future driver work. Do not
quote the total as detection coverage.

Known imperfections (not hidden): brand display names carry
source-spelling case variants (SerComm/Sercomm, CradlePoint/
Cradlepoint, Microtik/MikroTik, Banana Pi/Banana pi) because the
sweep preserves source spellings and dedupes on normalized keys; 789
models carry parenthetical text from source titles (e.g. OpenWrt
layout variants like "A10 (OpenWrt U-Boot layout)"); a few
title-fallback brands are junk-ish ("4G", "unknown"). A future
brand-normalization pass can clean these without changing profile IDs.
