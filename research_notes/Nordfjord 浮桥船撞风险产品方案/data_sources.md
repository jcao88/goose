# Data Sources for a Ship–Bridge Allision Risk and 3D Digital-Twin Product (Nordfjorden Anda–Lote case + Francis Scott Key Bridge validation)

Research notes as of 2026-10-19. How these notes were made: Norwegian government domains (kystverket.no, kystdatahuset.no, barentswatch.no, geonorge.no, data.norge.no, met.no, thredds.met.no, frost.met.no, fiskeridir.no) and ntsb.gov / noaa.gov **could not be fetched directly** from this environment. WebFetch failed on DNS and curl got a 403 from the proxy. Findings therefore come from (a) search-engine summaries of those official pages, (b) GitHub-hosted primary material that could be read in full: the NOAA MarineCadastre repo, the NOAA BlueTopo repo, MET Norway's `metocean-api` docs, and an open-source Kystdatahuset API client whose source code I inspected, and (c) secondary press coverage. The shared web-search budget ran out partway through. Some items I would normally have checked therefore sit under **Gaps**, or under **Inferences** marked "(background knowledge, not verified this session)". Re-check every licence and URL against the live portal before you build on it.

Case-context caveat. The searches did **not** confirm that an Anda–Lote floating bridge is currently planned or funded:
- In 2014 the government chose an inner E39 route through Stryn, with a suspension bridge at Svarstad ([TU 2014](https://www.tu.no/artikler/regjeringen-valgte-indre-e39-trase/196355)).
- A 2024 Ministry of Transport supplementary allocation letter is titled "krysning av Nordfjorden – bruk av ferjeavløsningsmidler". It says that ferry-replacement funds plus tolls alone cannot finance replacing national-road ferry routes ([Regjeringen 2024](https://cms12.regjeringen.no/contentassets/087954c97c8e47b7b2a733e350a25b4a/2024/statsbudsjettet-2024-supplerende-tildelingsbrev-nr.-15-krysning-av-nordfjorden-bruk-av-ferjeavlosningsmidler-mv.-l2425671.pdf)).
- The Anda–Lote ferry (about 11 min, E39, Gloppen–Stad) still runs ([Fjord1/Nordfjord](https://www.nordfjord.no/en/anda-lote-fjord1)).
- Statens vegvesen replaced the Lote ferry ramp in November 2025 ([Statens vegvesen 2025](https://www.vegvesen.no/om-oss/presse/aktuelt/2025/10/erstattar-ferjebru-pa-e39-lote-ferjekai/)).

Treat the "floating bridge at Anda–Lote" as a hypothetical or concept-stage design case.

---

## 1. AIS traffic data: Norway (Kystverket / Kystdatahuset / BarentsWatch) and the US (MarineCadastre), plus commercial and EMODnet

### Takeaway
For Nordfjorden the workable no-cost stack has three parts:
- **Kystverket HAIS**: historic positions from 2005, delivered as GeoParquet through an interactive order.
- **Kystdatahuset**: REST API with JWT login. It has passage lines, ship lookup and NOR VTS incidents.
- **BarentsWatch AIS API**: OAuth2, live data, with history capped at 14 days.

All open tiers are NLOD and leave out fishing vessels under 15 m and leisure craft under 45 m. Full data needs an application to Kystverket.

For Baltimore, NOAA MarineCadastre 2024 data is CC0. It has a daily GeoParquet file for the accident day (`ais-2024-03-26.parquet`), but it is downsampled to 1 minute.

Commercial AIS (Spire, now Kpler-related, MarineTraffic/Kpler, S&P) is priced only by quote.

### Cited Findings

**Kystverket HAIS (Historic AIS)**
- HAIS gives historic ship positions from the Norwegian AIS network, covering both base stations and Kystverket's polar-orbiting satellites. Coverage is the Norwegian EEZ plus the fishery zones around Jan Mayen and Svalbard. You filter by area, time period, single ship or ship type, and receive a download link to a GeoParquet file. The metadata lists GeoParquet 1.0.0, an access point at `hais.kystverket.no` and daily updates — [Geonorge HAIS record](https://geonorge.no/geonetworktest/srv/api/records/6a58c9e5-302c-454b-b229-2a317b1ac902)
- The archive goes back to 2005, with clearly better coverage from 2010 and again from 2016 — [Geonorge HAIS record](https://geonorge.no/geonetworktest/srv/api/records/6a58c9e5-302c-454b-b229-2a317b1ac902)
- Fishing vessels under 15 m and leisure vessels under 45 m are excluded from publication — [Geonorge HAIS record](https://geonorge.no/geonetworktest/srv/api/records/6a58c9e5-302c-454b-b229-2a317b1ac902)
- The metadata lists no API. The October 2026 GitHub issue below also describes HAIS as "an interactive order flow that emails a download link, so it is not automatable" — [Noesis issue #2851](https://github.com/Ikey168/Noesis/issues/2851)

**Kystverket open AIS (12 nm product)**
- A separate open product covers vessels over 45 m inside 12 nm of the baseline, as a CSV download from `ais-public.kystverket.no`. No API is registered for it — [data.norge.no](https://data.norge.no/en/datasets/03bdbcf2-ff09-4315-821c-eaa4711ac81f/automatisk-identifikasjonssystem-ais-shipsposisjoner-nedlasting-12nm-fra-grunnlinja)
- A third-party developer reported in October 2026 that `ais-public.kystverket.no` "did not respond from the server or a desktop" — [Noesis issue #2851](https://github.com/Ikey168/Noesis/issues/2851). This is one developer's report and is unverified.

**Licence and access tiers (Kystverket)**
- Open AIS data is free under the Norwegian Licence for Open Government Data (NLOD) and needs no registration. Access to *all* AIS data needs an application form that states the purpose. That data goes mainly to public authorities and ports, but others can apply — [Kystverket "Access to AIS data"](https://www.kystverket.no/en/sea-transport-and-ports/ais/access-to-ais-data/) (via search summary)

**Kystdatahuset (REST API; verified from client source code)**
- Base URL: `https://kystdatahuset.no/ws/api`. OpenAPI/Swagger: `https://kystdatahuset.no/ws/swagger/index.html` — [ai4copsec/kystdatahuset-apiclient](https://github.com/ai4copsec/kystdatahuset-apiclient)
- Auth: POST username and password to `/auth/login/`, which returns a JWT used as a Bearer token. The credentials are your Kystdatahuset/kystinfo.no account — [kystdatahuset-apiclient](https://github.com/ai4copsec/kystdatahuset-apiclient)
- Endpoints the client uses:
  - `ship/combined/mmsi/{mmsi}`: combined ship record
  - `ship/for-mmsis`: batch lookup
  - callsign lookup
  - `kystinfo/norvts-incidents`: NOR VTS incident reports filtered by start and end time
  - The client can also call any other endpoint as a raw request
  
  Source: [kystdatahuset-apiclient](https://github.com/ai4copsec/kystdatahuset-apiclient)
- A 2019 COWI ship-traffic analysis describes Kystdatahuset as a national ship-traffic database. You can draw a custom **passage line** in it and get every ship passage across that line — [COWI 2019 skipstrafikkanalyse (Vestfold)](https://vestfoldfylke.no/globalassets/bypakke-tonsberg-regionen/dokumenter/overordnet-styringsgruppe-dokumenter/2019/21-november/sak_27-19_vedlegg_rap-kon-001-vedlegg-2-skipstrafikkanalyse.pdf)
- An open-source well-boat study lists "Kystverket / Kystdatahuset – historical AIS data (ship movements, port calls)" under NLOD — [mariu1914/bronnbat-ais](https://github.com/mariu1914/bronnbat-ais)

**Kystverket official passage statistics**
- Kystverket publishes AIS-based passage counts per year and direction, by ship category and size, on about 30 predefined lines along the coast. Ships with AIS switched off are not counted — [Kystverket metadata: skipspasseringer](https://www.kystverket.no/contentassets/1faf2c86b6104e5e9ab5b4dfb0b289a6/metadata-om-statistikk-for-skipspasseringer.pdf/download)

**BarentsWatch AIS API**
- There is a Live AIS API (latest positions) and a track/historic API (a vessel's tracks for the last 24 h or a chosen period, at `historic.ais.barentswatch.no`). You register at barentswatch.no, create an API client on MyPage and request a token with scope `ais` — [BarentsWatch Live AIS docs](https://developer.barentswatch.no/docs/AIS/live-ais-api); [BarentsWatch blog 2022-05-31](https://developer.barentswatch.no/blog/2022/05/31)
- The historic service holds only the latest **14 days**, and the same small-vessel exclusions apply — [BarentsWatch AIS examples](https://developer.barentswatch.no/docs/AIS/examples)
- Token endpoint: `https://id.barentswatch.no/connect/token` (OAuth2 client credentials, `BARENTSWATCH_CLIENT_ID` / `BARENTSWATCH_CLIENT_SECRET`). The well-boat project also pulls weekly historical vessel tracks for wellboats through the Fishhealth API, `/bwapi/v1/geodata/fishhealth/vesseltrack/{mmsi}/{year}/{week}` — [bronnbat-ais source](https://github.com/mariu1914/bronnbat-ais)
- Kystverket says open data needs no registration, while the BarentsWatch API does need a registered client — [Kystverket access page](https://www.kystverket.no/en/sea-transport-and-ports/ais/access-to-ais-data/); [BarentsWatch docs](https://developer.barentswatch.no/docs/AIS/live-ais-api)

**Terrestrial vs satellite AIS (Kystverket)**
- Coastal base stations collect AIS. Satellites were launched to cover northern sea areas that land stations do not reach — [SNL: Norske satellittar](https://snl.no/Norske_satellittar)
- Satellite history:
  - AISSat-1 launched in 2010 (Kystverket/FFI). AISSat-1 and -2 have been retired.
  - AISSat-3 was lost in a launch failure.
  - The NorSat series (NorSat-1/-2/-3/-TD, and NorSat-4) carry AIS for Kystverket.
  - AISSat-4 is reported launched on 30 March 2026, to replace capacity lost when several NorSats went out of service.
  
  Source: [SNL](https://snl.no/Norske_satellittar). Mixed-date and single-source. The summary's "NorSat-1/-2 launched 2014" conflicts with the July 2017 launch date I remember, so verify it.
- Satellite data is downlinked at Vardø and Svalbard, and its coverage is intermittent — [TU](https://www.tu.no/artikler/nytt-satellitt-senter-i-vardo-gir-sikrere-skipstrafikk-i-arktis/223378); [Kystverket NorSat news](https://www.kystverket.no/en/News/norways-new-satellite-detects-radar-signals-from-ships)

**US: NOAA MarineCadastre / AccessAIS (USCG NAIS)**
- Data comes from USCG Nationwide AIS, hosted by NOAA Marine Cadastre. Access routes:
  - **AccessAIS**: custom area and time, zipped CSV, orders up to about 2 GB (`https://marinecadastre.gov/accessais/`)
  - **Bulk annual files** from 2009 (`https://hub.marinecadastre.gov/pages/vesseltraffic`). Older years are geodatabase, newer years CSV.
  - **Experimental GeoParquet**: 2024 daily broadcast points at `https://ocmgeodatastor1.blob.core.windows.net/marinecadastre/ais2024/`, and 2024–2025 monthly tracks at `.../marinecadastre/aistrack/`
  - The GeoParquet files had extra cleaning to remove sentinel values and anomalies found in AccessAIS. Licence is **CC0 1.0**.
  
  Source: [ocm-marinecadastre/ais-vessel-traffic](https://github.com/ocm-marinecadastre/ais-vessel-traffic)
- 2024 broadcast points: GeoParquet 1.0.0, Snappy compression, WKB Point geometry, WGS84, UTC, **downsampled to the nearest whole minute**, daily files of up to about 325 MB — [broadcast-points readme](https://github.com/ocm-marinecadastre/ais-vessel-traffic/blob/main/data/ais-broadcast-points-2024-readme.md)
- 2024 tracks: GeoParquet 1.1.0, ZSTD compression, LineString geometry, 1-minute, monthly files of up to about 1.78 GB. Citation: Martin et al. 2026, NOAA OCM — [tracks readme](https://github.com/ocm-marinecadastre/ais-vessel-traffic/blob/main/data/ais-tracks-2024-readme.md)
- The file list includes `ais-2024-03-25.parquet`, `ais-2024-03-26.parquet` (the allision day) and `ais-2024-03-27.parquet` — [daily file list](https://github.com/ocm-marinecadastre/ais-vessel-traffic/blob/main/data/daily-2024-parquet-filenames.md)
- The NAIS 2024 InPort record says the source records were filtered to a 1-minute rate and limited to the US EEZ — [NOAA InPort 73064](https://www.fisheries.noaa.gov/inport/item/73064)
- AIS Vessel Transit Counts are available as ArcGIS REST tile services for 2020–2024 (for example `https://coast.noaa.gov/arcgis/rest/services/MarineCadastre/AISVesselTransitCounts2024/MapServer`) — [MarineCadastre repo](https://github.com/ocm-marinecadastre/ais-vessel-traffic); [InPort 75938](https://www.fisheries.noaa.gov/inport/item/75938)

**Commercial AIS**
- Spire Maritime historical AIS: quote-based pricing, either a yearly subscription to the historical APIs or a one-off purchase priced by your sample settings. There is an online shop at `historical.ais.spire.com` — [Spire Historical AIS](https://spire.com/?p=7420); [historical.ais.spire.com](https://historical.ais.spire.com)
- MarineTraffic (owned by Kpler) runs a freemium model with paid subscriptions and API. Kpler markets historical AIS for academic research but publishes no university rate — [Kpler Historical AIS](https://www.kpler.com/product/maritime/historical-ais)
- A July 2026 MarineTraffic update claims 18× more area coverage per API query and full-year tracks for 100 vessels in one call. This is a vendor claim — [Hellenic Shipping News, July 2026](https://hellenicshippingnews.com/from-dark-port-calls-to-dead-zones-julys-marinetraffic-updates)
- The only published price table found was AIS Hub, a different and community-type provider, at $0–$300 per month — [apispine AIS Hub pricing](https://apispine.com/ais-hub/pricing)

**EMODnet Human Activities vessel density**
- 1 km × 1 km grid (EEA/INSPIRE grid) over EU waters and some neighbouring areas, in hours per km² per month by ship type (categories 0–12, including "All"). It is built from AIS bought from CLS and ORBCOMM (terrestrial plus satellite) and updated yearly. The method draws a line between consecutive positions and gives segment durations to the cells they cross — [EMODnet detailed method](https://emodnet.ec.europa.eu/en/human-activities-eu-vessel-density-map-detailed-method); [EMODnet blog Dec 2025](https://emodnet.ec.europa.eu/en/blog-8-december-2025-behind-scenes-emodnet-human-activities-vessel-density-maps); [Method PDF 2023](https://emodnet.ec.europa.eu/sites/emodnet.ec.europa.eu/files/public/HumanActivities_20231101_VesselDensityMethod.pdf); [OSPAR](https://odims.ospar.org/en/submissions/ospar_vessel_density_2019_03/)
- The original 2019 release covered 2017 and referred to 14 ship categories. Category counts therefore differ between versions — [EMODnet method](https://emodnet.ec.europa.eu/en/human-activities-eu-vessel-density-map-detailed-method)

### Inferences
- **How AIS feeds the risk model.** The core model follows AASHTO Method II or IALA/Fujii–Pedersen category I/II:
  - Draw a passage line along the proposed bridge axis between Anda and Lote, in Kystdatahuset or HAIS.
  - From it, derive annual transits *N* by ship type and size bin, and the lateral distribution of transits across the fjord (fit normal or mixture distributions for Pedersen category I, ships on normal course).
  - Get speed distributions per class, which drive impact energy.
  - Get heading and course-change points upstream, which feed category II (failure to turn) and the 3D replay.
  - The ship-type mix over time matters. Cruise traffic to Olden, Loen, Stryn and Nordfjordeid would drive design-energy scenarios.
- **Sampling.** 1-minute data (MarineCadastre, and possibly the Norwegian downsampled products) is enough for lateral distributions and counts. It is **not** enough to reconstruct a blackout-to-impact sequence that lasts only a few minutes; for that, use the NTSB docket/VDR (see section 7). I could not confirm the native rate in HAIS; raw Class A AIS reports every 2–10 s under way (background knowledge, not verified this session).
- **Small-craft exclusion.** Under the open licence, leisure craft under 45 m and fishing vessels under 15 m are missing. That barely matters for structural design energy but does matter for navigation-span and pontoon-contact frequency and for a "complete traffic picture" in the digital twin. A Kystverket full-access application (purpose: bridge risk research) is the clean fix.
- **Satellite AIS is mostly irrelevant inside Nordfjorden.** Terrestrial base-station coverage decides data quality in an inner fjord. Check for coverage shadows behind steep terrain by plotting message gaps.
- **Commercial AIS** is only needed for (a) global ship-behaviour priors, (b) cross-checking Norwegian gaps, or (c) automated, licence-clean redistribution in a commercial SaaS. Kystverket's restricted tier may not permit (c).
- **EMODnet** 1 km cells are coarse relative to the width of Nordfjorden at Anda–Lote. The ~11-minute ferry suggests a crossing on the order of a few km; I did not verify this. Use EMODnet only as a regional context layer or for multi-site screening across European bridges, not for the site-specific lateral distribution.

### Gaps
- I could not open the Kystdatahuset Swagger page, so I could not confirm the AIS position and passage-line endpoints, rate limits, account eligibility (whether non-public-sector users get accounts), its licence (NLOD or a separate agreement), or its archive depth.
- HAIS: native sampling rate, schema, per-order size limits and whether it covers years before 2010 at useful density.
- BarentsWatch: whether the open AIS licence permits commercial redistribution beyond NLOD, and rate limits.
- Commercial AIS cost ranges: no published prices found. The S&P Global AIS offering was not researched because the search budget ran out. Spire Maritime's acquisition by Kpler is background knowledge, not verified this session.
- No Kystverket statement found on terrestrial coverage quality specifically in inner Nordfjord.

---

## 2. Ship static and parameter data (what AIS lacks, and where to get it)

### Takeaway
AIS static messages give MMSI, IMO, call sign, name, ship type, length/beam (GPS antenna offsets), draught and destination. They do **not** give displacement, deadweight, air draft, bow shape or structural stiffness. Kystdatahuset's `ship/combined` endpoint can enrich Norwegian-observed MMSIs. Equasis (free registration) or S&P Sea-web (paid) provide DWT and GT. Displacement, air draft and bow geometry have to come from type-specific regressions, cruise spec sheets or design codes. I found no single open "air-draft by ship type" dataset.

### Cited Findings
- Kystdatahuset exposes ship look-ups by MMSI and call sign (`ship/combined/mmsi/{mmsi}`, `ship/for-mmsis`) returning a combined ship record — [kystdatahuset-apiclient source](https://github.com/ai4copsec/kystdatahuset-apiclient). The fields are not documented in the client.
- Kystverket's Havbase links AIS data with ship-register information on engine technology to estimate emissions. This shows Kystverket internally joins AIS with ship-register technical data — [Kystverket metadata: driftsutslipp til luft](https://www.kystverket.no/contentassets/456c82ae26df47d9b892c610e72f8493/metadata-om-statistikk-for-driftsutslipp-til-luft.pdf/download)
- The well-boat study used Sjøfartsdirektoratet's ship register for vessel category (well-boat flag). No licence was stated — [bronnbat-ais README](https://github.com/mariu1914/bronnbat-ais)
- The Bjørnafjorden E39 floating-bridge work treated **cruise ships** on regular routes past the bridge as a key accidental-collision concern. An FE study of a cruise ship striking a steel box girder found the girder design generally safe against normal accidental collision loads — [Structurae: Design of floating bridge girders against accidental ship collision loads](https://structurae.de/de/fachliteratur/tagungsbeitrag/design-of-floating-bridge-girders-against-accidental-ship-collision-loads); [NTNU Open](https://ntnuopen.ntnu.no/ntnu-xmlui/handle/11250/2563974)

### Inferences
(background knowledge, not verified this session)
- **Equasis** (equasis.org): free registration. It gives IMO-level particulars (type, flag, GT, DWT, build year), manager/owner, PSC inspections and class. Its terms forbid bulk scraping and redistribution, so use it for manual or small-batch enrichment of the few hundred distinct large ships that cross Anda–Lote each year.
- **S&P Global Sea-web** (formerly IHS Fairplay) is the industry reference for full particulars, including displacement where available, depth, and sometimes air draft. It is a paid subscription, typically institutional and quote-based. Universities can sometimes get it through library licences. **Clarksons World Fleet Register** is an alternative.
- **Norwegian Ship Registers (NOR/NIS)**, run by Sjøfartsdirektoratet, are useful for the local fleet: ferries, fast ferries, well-boats and fishing vessels. **Fiskeridirektoratet's merkeregister** (fishing-vessel register) is open and gives length, GT and engine power for fishing vessels.
- **Displacement and mass**:
  - The AASHTO Guide Spec (1991/2009) and Eurocode EN 1991-1-7 Annex C give design ship classes with DWT and displacement.
  - Typical regressions are Δ ≈ DWT / C_DWT, with C_DWT around 0.75–0.85 for tankers and bulkers, around 0.6–0.75 for container ships and much lower for cruise/RoPax. Alternatively Δ = ρ·Cb·L·B·T, using the AIS draught and Cb by type.
  - Added mass coefficients (about 0.05–0.10 surge, 0.4–1.0 sway) come from the Pedersen/Zhang analytical collision literature.
- **Air draft** matters for deck and girder strikes on a floating bridge with a high-rise navigation span. Practical sources: cruise-line and port specifications (Nordfjord ports publish maximum dimensions), class-society or shipyard spec sheets, and port air-draft restriction lists. A simple product approach is an air-draft-to-LOA regression per type, calibrated on about 50 cruise ships and container ships with known values.
- **Bow shape** (bulbous or raked, bulb geometry) drives the local crushing force on pontoons. It is not available in any open database. Use parametric bow families by type and size, as in DNV-RP-C204 and the Pedersen 1993/2010 bow-force formulas, and treat it as an uncertainty variable.

### Gaps
- I did not verify Equasis terms, Sea-web pricing, or whether the Kystdatahuset `ship/combined` response includes DWT, GT or air draft. The search budget ran out before these lookups.
- I found no peer-reviewed open dataset or regression specifically for **air draft vs ship type/size**. This needs a targeted literature search, for example port air-draft studies for the Øresund and Great Belt bridges or the US Bayonne Bridge.
- Sdir's ship register access method (API or download) and licence were not confirmed.

---

## 3. Met-ocean data (wind, waves, currents, tide, visibility)

### Takeaway
MET Norway's data is all open (NLOD/CC BY 4.0) and reachable through THREDDS/OPeNDAP:
- **NORA3** (3 km HARMONIE-AROME atmospheric hindcast; companion 3 km WAM wave hindcast; coastal **NORAC** wave product): wind, wave and drift climatology.
- **NorKyst-800**: archive from 2016-09-14; v3 adds 160 m two-way-nested fjord domains. Used for currents.
- **Kartverket's vannstand API** (now at `vannstand.kartverket.no`): tides.
- **Frost API** (free client ID): station observations. **Sandane Airport (ENSD), located at Anda itself**, issues METARs, a strong source for visibility and fog at the bridge site.

### Cited Findings

**NORA3 atmosphere**
- Non-hydrostatic HARMONIE-AROME at 3 km horizontal resolution with 65 vertical levels, forced by ERA5. It runs 9-hourly forecasts four times a day — [NORA3 paper (BORA/UiB)](https://bora.uib.no/bora-xmlui/handle/11250/2998467); [WindPRO NORA3 wiki](https://help.emd.dk/mediawiki/index.php/NORA3)
- Sources disagree on the period covered:
  - The data.met.no atmosphere record lists 1985-01-01 to 2021-12-31: [data.met.no NORA3 atmosphere](https://data.met.no/dataset/64636e8c-c486-4496-bdda-89687e1d8f97)
  - The hourly wind dataset lists a 1970 start: [data.met.no NORA3 hourly wind](https://data.met.no/dataset/ddb4c5d1-70b5-4d09-98a8-fadbf5e2d2ba)
  - ocean.met.no says "from 1960": [ocean.met.no waves](https://ocean.met.no/waves)
  - A 2024 study used 1982–2022: [Renewable Energy 2024](https://ideas.repec.org/a/eee/renene/v224y2024ics0960148124002490.html)
  
  The archive has evidently been extended over time.

**THREDDS catalogue paths (from MET-OM `metocean-api` docs)**
- NORA3 hourly wind at 10–750 m: `https://thredds.met.no/thredds/catalog/nora3_subset_atmos/wind_hourly_v2/catalog.html`
- NORA3 hourly atmosphere: `.../nora3_subset_atmos/atm_hourly/`
- NORA3 3-hourly: `.../nora3_subset_atmos/atm_3hourly/`
- NORA3 raw output: `https://thredds.met.no/thredds/projects/nora3.html`
- NORA3 wave subset: `.../nora3_subset_wave/wave_tser/`
- NORA3 3 km WAM wave fields: `.../windsurfer/mywavewam3km_files/`
- NORA3 wave spectra: `.../windsurfer/mywavewam3km_spectra/`
- NORA3 storm surge: `.../stormrisk/`
- **NORAC coastal wave**: `.../norac_wave/field/`, spectra at `.../norac_wave/spec/`
- **NorKyst800** (from 2016-09-14 to today): `https://thredds.met.no/thredds/fou-hi/norkyst800v2.html`
- NorkystDA 2017–2018: `.../nora3_subset_ocean/`
- **Statens vegvesen E39 fjord wave buoys**: `https://thredds.met.no/thredds/catalog/obs/buoy-svv-e39/catalog.html`
- Python access: `pip install metocean-api`, e.g. `ts.TimeSeries(lon, lat, start_time, end_time, product='NORA3_wind_wave')`
- Parallel downloading of NORA3 breaches the thredds.met.no terms of service and can get you blocked

Source: [metocean-api docs](https://github.com/MET-OM/metocean-api/blob/main/docs/index.rst)

**NorKyst (currents)**
- NorKyst-800 is ROMS-based on an 800 m grid. It is forced by AROME 2.5 km atmosphere, Nordic-4km boundaries and TPXO 7.2 tides, and forecasts to +66 h — [ROMS 2014 poster (Ådlandsvik)](https://myroms.org/Workshops/ROMS2014/presentations/Posters/Adlandsvik.pdf)
- The **NorKyst v3** paper describes horizontal resolution from 2.4 km (analysis) to 800 m (forecast), with **160 m** in two-way nested fjord domains. The catalogue is at `thredds.met.no/thredds/fou-hi/norkystv3.html` — [Ocean Dynamics 2020](https://link.springer.com/article/10.1007/s10236-020-01378-0) (via search summary); [ocean.met.no](https://ocean.met.no/node/4)
- The NorKyst800 forecast is a rolling archive in which only recent data stay online — [Geonorge NorKyst800 forecast](https://geonorge.no/geonetworktest/srv/api/records/ADC_ROMS-NorKyst800-forecast)
- The Institute of Marine Research (IMR) runs a free web tool, the "Norwegian Current Information System", with archives of model currents for coastal and fjord areas (Norwegian UI) — (search summary of NorKyst sources; [ocean.met.no](https://ocean.met.no/node/4))

**Fjord waves downscaling**
- The open NORA-SARAH framework (arXiv 2509.21329, 2025) uses NORA3/NORA10EI as offshore boundary conditions, propagates them nearshore with SWAN, then feeds phase-resolving and CFD solvers. It is a ready template for fjord wave and current loads on pontoons — [arXiv 2509.21329](https://arxiv.org/pdf/2509.21329)
- MET Norway has applied SWAN to fjord hindcasts for the E39 project and validated them against buoys — [MET report 03/2021](https://www.met.no/publikasjoner/met-report/_/attachment/download/424fe337-b444-4801-913d-ba7207a29024:801be70d4b238debdf6abbfb897273e89304b6ae/MET-report_03-2021.pdf)
- NORA10 is a ~10 km hindcast covering 1957–2022 — [ocean.met.no waves](https://ocean.met.no/waves) (via search summary)

**Tides (Kartverket)**
- The Water Level API has moved from `api.sehavniva.no` to **`vannstand.kartverket.no`**, HTTPS only; the old host lost support after August 3. It returns observed water level, astronomical tide predictions and forecasts, for a permanent tide gauge or **any position**; position queries use a tidal-zone model of the coast. Output is XML or a file. Values are in cm relative to chart datum. Free products are CC BY 4.0. Contact: tidevann@kartverket.no — [Geonorge notice](https://register.geonorge.no/varsler/vannstand-api/2ede2455-47e0-49ff-9191-68f598429ab0); [API protocol rev. June 2025](https://vannstand.kartverket.no/API%20for%20water%20level%20and%20tides%20-%20communication%20protocol_revJune2025.pdf); [opencom dataset](https://opencom.no/dataset/vannstand-stavanger-kartverket)

**Station observations (Frost API)**
- Frost needs a free client ID, registered at frost.met.no. Data are NLOD 2.0 / CC BY 4.0 with attribution to MET Norway. Time resolutions run from 10-minute and hourly to daily, monthly and annual — [wetterdienst Frost docs](https://wetterdienst.readthedocs.io/en/latest/data/provider/metno/frost/index.html); [frostr README](https://cran.r-project.org/web/packages/frostr/readme/README.html)
- **Sandane Airport (ICAO ENSD) is at Anda.** Its METARs and TAFs are distributed internationally, for example the NOAA decoded feed, and historical METAR climatology exists for 2024 — [NOAA tgftp ENSD](https://tgftp.ncep.noaa.gov/data/observations/metar/decoded/ENSD.TXT); [metarcentral ENSD](https://metarcentral.com/airport/ENSD); [WeatherSpark 2024 Sandane/Anda](https://weatherspark.com/h/y/149963/2024/Historical-Weather-during-2024-at-Sandane-Anda-Norway)

### Inferences
- **How met-ocean feeds the model.**
  - (1) Visibility and fog frequency come from ENSD METAR and Frost. They condition the human-error causation probability Pc, for example a factor for reduced visibility on the IALA/Fujii base value of about 1–3×10⁻⁴.
  - (2) Wind speed and direction joint distributions come from NORA3 hourly wind at the bridge axis. They give drift direction and speed for blackout (category II) scenarios and a windage factor for high-sided cruise ships.
  - (3) Currents come from NorKyst-800, or the 160 m nest if one covers Nordfjorden. They give drift contribution and impact-angle distributions.
  - (4) Waves come from NORAC or SWAN downscaling. In an inner fjord they matter mainly for pontoon motion and the bridge's dynamic state at impact, not for collision frequency.
  - (5) Tide range comes from the vannstand API position query. It changes vertical clearance under the navigation span and the impact point on pontoons.
- **Resolution caveat.** A 3 km NORA3 grid only partly resolves fjord wind channelling in a fjord a few km wide with steep sides. Check NORA3 against Frost/ENSD stations, and consider MET's operational 2.5 km MEPS/AROME archive or local WRF/LES runs. MEPS on thredds.met.no is background knowledge, not verified this session.
- **NORA3 wave (3 km WAM)** will not resolve the Anda–Lote section; use NORAC or SWAN nesting instead.
- **ENSD visibility** is the best on-site fog proxy. Airport visibility is at the aerodrome, not mid-fjord, so treat it as an approximation.
- **NorKyst-800 archive** starts in September 2016, which gives about 10 years of currents. That is fine for current statistics; for the 50- to 100-year design return periods used in structural load analysis, combine it with NORA3.

### Gaps
- I could not confirm that a **"NorFjords-160"** product exists under that name, or that a 160 m nest covers Nordfjorden. Check the THREDDS catalogue `fou-hi`.
- NORAC wave resolution, model and period were not confirmed. No source was found.
- The Frost station IDs for Sandane/Anda and the nearest stations to Lote and Nordfjordeid, and whether visibility (for example `visibility_in_air` or `mor_visibility`) is archived in Frost, were not confirmed.
- I found no published study of **wind channelling in Nordfjorden specifically**.
- The nearest permanent Kartverket tide gauge to Anda–Lote was not confirmed; Måløy is plausible (background knowledge, not verified this session).

---

## 4. Bathymetry, terrain, imagery and charts for the 3D scene (Norway and Baltimore)

### Takeaway
Norway:
- Terrain is excellent and open: **Høydedata DTM1 (1 m)** and DTM10, CC BY 4.0, through hoydedata.no or Geonorge.
- Open bathymetry **inside the 12 nm/baseline zone is capped at a 50 m grid**, CC BY 4.0. Finer 5 m and 25 m grids are security-classified inside 12 nm. Non-partners request data at sjodata@kartverket.no.
- **Official ENC (S-57) data is restricted and needs an agreement with Kartverket**, while generalised depth contours are open (CC BY 4.0, S-57).

Baltimore:
- **NOAA BlueTopo** gives CC0 GeoTIFF bathymetry with vertical uncertainty from a public S3 bucket, plus a Python fetcher.

### Cited Findings

**Kartverket depth data**
- 50 m × 50 m terrain models of the seabed, CC BY 4.0, downloadable as TIFF and as NED/ENH/XYZ files. The national ("landsdekkende") 50 m grid comes through the Geonorge shopping cart. It is **interpolated, so reliability in position and depth varies**. Most high-quality coverage is in the north, near-continuous north of 67°N — [data.norge.no 50 m grid](https://data.norge.no/en/datasets/cf58d7fd-aeb6-4eb6-8bf0-0b8449b6c4e6/dybdedata-terrengmodeller-50-meters-grid); [Geonorge 50 m record](https://geonorge.no/geonetworktest/srv/api/records/67a3a191-49cc-45bc-baf0-eaaf7c513549)
- Inside the 12 nautical-mile limit, finer grids (5 m and 25 m) are generally classified. The WMS shows 25 m and 5 m only outside 12 nm, because depth data inside 12 nm that are more detailed than 50 m are classified — [INSPIRE Geoportal](https://inspire-geoportal.ec.europa.eu/srv/api/records/67a3a191-49cc-45bc-baf0-eaaf7c513549); [data.norge.no depth WMS](https://data.norge.no/en/data-services/248a12e4-0ec0-3c02-b3dc-21958549b6e0)
- Kartverket's product sheet says the public depth data are "ugraderte og til fri bruk" (unclassified, free use), and that higher-resolution data must be declassified before release. Point spacing follows Forsvarets navigasjonsplan, with 50 m between depth points — [Produktark sjøkart dybdedata 2021](https://dokument.geonorge.no/produktark/dybdedata/20210204/produktark-sjøkart-dybdedata-kartverket-20210204.pdf)
- Some Geonorge records say **non-partners must request data by email at sjodata@kartverket.no**, which may conflict with the open-download listings — [Geonorge depth record](https://geonorge.no/geonetworktest/srv/api/records/bbd687d0-d34f-4d95-9e60-27e330e0f76e)
- SeaDataNet lists a combined Kartverket 50 m grid under CC BY 4.0 — [SeaDataNet CDI](https://cdi.seadatanet.org/report/3382269/v0)
- Kartverket's 2024 allocation letter asks it to keep developing "Nautilus" for better depth-data services — [Tildelingsbrev 2024](https://cms12.regjeringen.no/contentassets/3617cd0bc4c8495da3b76beb0143d1a4/2024_tildelingsbrev_kartverket.pdf)

**Charts (ENC)**
- "Sjøkart – elektroniske (ENC)" is listed with **restricted access**: S-57 through a separate download page, licence not provided. "Elektroniske sjøkartdata" is for general, non-SOLAS use, cannot replace official ENCs, and producers who show it in their products need **an agreement with Kartverket** ("Egen avtale med Kartverket") — [data.norge.no ENC](https://data.norge.no/en/datasets/57255715-714f-36d7-90c8-b02263652083/sjokart-elektroniske-enc); [Geonorge electronic chart data](https://geonorge.no/geonetworktest/srv/api/records/0d05dc08-6da5-4d84-b31f-db3114c475f6)
- "Dybdedata – kurver generaliserte" (generalised depth contours) is distributed as S-57 under **CC BY 4.0** — [data.norge.no depth curves](https://data.norge.no/en/datasets/0e7e0dff-c86a-4da9-91b0-8598a5667f83/dybdedata-kurver-generaliserte)

**Terrain (Høydedata)**
- DTM1 (1 m) is **CC BY 4.0**. It is generated from point clouds held at hoydedata.no, downloaded from hoydedata.no or Geonorge bulk download in 15 km tiles, and a seamless WMS was published 2026-02-05 — [data.norge.no DTM1](https://data.norge.no/en/datasets/1a7327eb-1fa5-3432-8dea-fc198a5ede13/hoyde-dtm1); [data.norge.no DTM service](https://data.norge.no/en/data-services/df1adc4b-7575-351a-b9d4-4891decc8b16)
- 10 m DTMs are also generated from the same point clouds — [Geonorge DTM WMS record](https://geonorge.no/geonetworktest/srv/api/records/ff2b9b80-70b4-4280-95a5-19ed679d6933)

**US (Baltimore / Patapsco River)**
- **BlueTopo** (NOAA National Bathymetric Source) compiles the best available bathymetry for US waters from NOAA and USACE surveys plus topo-bathy lidar. It comes as multiband GeoTIFFs with **vertical uncertainty and source-survey bands**. BlueTopo is on a national vertical datum; the "Modeling", BAG and S-102 sources are on low-water datum. Hosted on a public S3 bucket. Python package `nbs.noaabathymetry`: `fetch_tiles(project_dir, geometry=bbox|gpkg|wkt|geojson)` and `mosaic_tiles()`. Licence CC0 — [NOAA BlueTopo](https://www.nauticalcharts.noaa.gov/data/bluetopo.html); [noaa-ocs-hydrography/BlueTopo](https://github.com/noaa-ocs-hydrography/BlueTopo)
- Official S-102 data is at `noaa-s102-pds.s3.amazonaws.com`; the package's S-102 versions are test-and-evaluation only — [BlueTopo repo](https://github.com/noaa-ocs-hydrography/BlueTopo)
- NCEI **CUDEM** has a Chesapeake Bay 1/3-arc-second topobathy tile set. The visible listing covers about 37–39°N; Baltimore at ~39.2°N needs checking — [NOAA CUDEM Chesapeake STAC](https://coast.noaa.gov/htdata/raster2/elevation/NCEI_third_Topobathy_2014_8580/stac/chesapeake_bay)
- Maryland iMAP Chesapeake bathymetry contours come from 297 surveys dated 1859–1993 (about 3.18 M soundings). They are dated, so use only as background — [data.gov MD iMAP](https://catalog.data.gov/dataset/md-imap-maryland-bathymetry-chesapeake-bay-contours)

### Inferences
- **How it feeds the 3D scene and the model.**
  - Høydedata DTM1 provides fjord-side terrain meshes, converted to quantized-mesh or 3D Tiles for Cesium.
  - The Kartverket 50 m grid, merged with the DTM at the shoreline, provides the seabed. It also sets the navigable-water polygon for drift and grounding-before-allision calculations: a ship that grounds before reaching the bridge is a risk reducer in the Pedersen model.
  - The 50 m grid is coarse but acceptable inside a deep fjord (Nordfjorden is deep, so grounding barriers matter mainly near the shores and landings).
  - Pontoon and anchor-line design would need the NPRA's own multibeam survey, which is likely classified or project-restricted.
- **ENC.** For a commercial product, do not ship Kartverket ENC content without an agreement. Use the open generalised depth contours plus your own rendering of fairway and aids-to-navigation data from Kystverket (see section 6). The IHO S-101 transition is ongoing; whether Norway offers S-101 production ENCs in 2026 is unverified.
- **Imagery.** Norge i bilder orthophotos (Geovekst) are commonly viewable through WMS for non-commercial use. Downloading or commercial use usually needs Norge digitalt membership or an agreement (background knowledge, not verified this session). For a 3D demo, alternatives are Sentinel-2 (10 m, free), Cesium ion's Bing or Google imagery under their terms, or Esri World Imagery.
- **Cesium.** Cesium ion offers Cesium World Terrain, Cesium World Bathymetry (GEBCO-based, coarse), Google Photorealistic 3D Tiles and OSM Buildings, each under its own ion or Google terms (background knowledge, not verified this session). For Nordfjord, self-hosted 3D Tiles from DTM1 plus the 50 m bathymetry give far better fidelity and cleaner licensing (CC BY 4.0 attribution) than the ion global products.
- **Baltimore.** BlueTopo plus USGS 3DEP lidar (background knowledge) plus the free NOAA ENCs (background knowledge) give a fully CC0/public-domain 3D scene. The bridge structure itself (spans, pier positions, dolphin and fender locations) must be modelled from NTSB report figures or MDTA drawings.
- **EMODnet Bathymetry DTM** (~1/16 arc-min, about 115 m, CC BY 4.0; background knowledge) is only a fallback where Kartverket's grid has holes.

### Gaps
- No confirmed source on any 2024–2026 relaxation of Norwegian depth-data classification inside the baseline. A narrower search on the Forsvaret / "gradering dybdedata" announcements is needed.
- Norge i bilder licence terms for commercial 3D use were not confirmed.
- I did not verify Cesium ion bathymetry and Google 3D Tiles coverage or terms for Nordfjord.
- I did not verify Kartverket's current S-101 / S-102 availability for Norwegian waters.

---

## 5. Accident and incident data for causation-probability calibration

### Takeaway
Norway:
- **Sjøfartsdirektoratet's accident database** (the successor that merged legacy **DAMA** 1981–2006 and PUS) holds over 32,000 accidents, about 8,900 of them ship accidents. I found **no public bulk download**. Request it from Sdir, or use SSB's derived statistics.
- Kystdatahuset exposes **NOR VTS incident reports** through its API. The fields include event type, ship type, size, wind, MMSI and IMO, which suits near-miss and causation calibration.

US and EU:
- **USCG MISLE** "Marine Casualty and Pollution Data for Researchers" is downloadable and includes the allision type.
- **EMSA EMCIP** is available only as annual aggregates for non-member-state users.

I found **no published statistics on ship–bridge allisions in Norway for 2018–2026**.

### Cited Findings
- **Sdir database.** Sjøfartsdirektoratet records ship accidents and personal accidents on vessels. DAMA (1981–2006) and PUS (1989–2006) were merged into one accident database in 2006, which holds over 32,000 accidents, including about 8,900 ship accidents. A risk analysis used full ship-accident data for 1981–2018 — [Sdir "Fra historiske data til fremtidig estimat"](https://www.sdir.no/contentassets/cd377adee098411cbb8c47963c7df471/fra-historiske-data-til-fremtidig-estimat.pdf) (via search summary)
- Kystverket's 2014 maritime safety analysis says around **100 groundings per year** have been registered in Norwegian waters since 2007, based on the Sdir database — [Kystverket Sjøsikkerhetsanalysen 2014 – ulykkesanalyse](https://kystverket.no/contentassets/581540c2ad1e4543a4d10d31decab137/2_ulykkesanalyse.pdf)
- **SSB** publishes ship-accident statistics derived from Sdir data, released within six months of the period — [SSB sjøulykker (om statistikken)](https://www.ssb.no/transport-og-reiseliv/statistikker/sjoulykker/aar/2012-07-27?fane=om)
- Sdir also publishes risk-picture reports ("Fokus på risiko") — [Sdir Fokus på risiko 2018](https://www.sdir.no/globalassets/sjofartsdirektoratet/fartoy-og-sjofolk---dokumenter/roh---dokumenter/fokus-pa-risiko-2018-norsk.pdf)
- **NOR VTS incidents (Kystdatahuset API `kystinfo/norvts-incidents`)**. The record schema includes:
  - `id`, `lat`, `lng`
  - `hendelsesid`, `hendelsetype` (event type), `dato`
  - `skipstype`, `storrelse_bt` (size in BRT)
  - `merknad` (context comments), `registrert_av` (source)
  - `link_news`, `open_media`
  - `vindretning`, `vindstyrke` (wind direction and strength)
  - `mmsi`, `imo`, `ship_name`, `skipstype_skipsregister`
  
  It can be filtered by time window and exported to JSON or Parquet — [kystdatahuset-apiclient `core/config.py` Incident model](https://github.com/ai4copsec/kystdatahuset-apiclient)
- **NSIA (Statens havarikommisjon)** publishes marine investigation reports in English at nsia.no, for example the Helge Ingstad / Sola TS collision part-two report (2021) — [NSIA report 2021/05](https://nsia.no/Marine/Published-reports/2021-05-eng)
- Searches found **no NSIA report of a ship striking a bridge in Norway**. They returned only collisions and groundings: Helge Ingstad 2018, NCL Salten grounding May 2025, and a three-ship collision near Ålesund in October 2025 — [Maritime Executive](https://maritime-executive.com/article/norwegian-authorities-investigating-collision-of-three-ships); [CBS](https://www.cbsnews.com/news/norway-ship-grounded-cabin-helmsman-asleep-reports/)
- A nearby Scandinavian precedent: the timber carrier **Mirabelle** was reported to have struck Denmark's Great Belt Bridge (about 2005), and the officer on watch, who had fallen asleep, was killed — [gCaptain](https://gcaptain.com/vessel-history-incidents-sheet/)
- **EMSA EMCIP** is the EU's central database of marine casualties and incidents. EMSA publishes an Annual Overview by taxonomy: the 2025 edition covers 2015–2024, with 2,659 casualties and incidents in 2024, and very serious casualties at 52 (2020), 61 (2021), 57 (2022) and 45 (2023). Scope is EU-flag ships, EU territorial and internal waters, or other substantial EU interests. EMSA warns the data do not show the full picture and that reporting lags. Norway is an EEA member; its participation in EMCIP is background knowledge, not verified this session — [World Ports / EMSA 2025](https://www.worldports.org/?p=633056); [IIMS on EMSA 2024](https://iims.org.uk/emsa-annual-overview-of-marine-casualties-and-incidents-2024)
- **USCG MISLE** "Marine Casualty and Pollution Data for Researchers" can be downloaded from USCG Homeport/DCO. Casualties go back to 1982 and pollution to 1973. It comes as zipped files with a record-layout readme; the historical extract runs to July 2015, and the current release status is unconfirmed. **Allision** is a listed casualty type — [data.gov](https://data.gov/maritime/safety-at-sea-us-coast-guard-marine-casualty-and-pollution-data-for-researchers/); [USCG DCO page](https://www.dco.uscg.mil/Our-Organization/Assistant-Commandant-for-Prevention-Policy-CG-5P/Inspections-Compliance-CG-5PC-/Office-of-Investigations-Casualty-Analysis/Marine-Casualty-and-Pollution-Data-for-Researchers); [MISLE readme](https://www.dco.uscg.mil/Portals/9/DCO%20Documents/5p/CG-5PC/INV/docs/MisleReadme.docx); [Delaware Bay response annex](https://nrt.response.epa.gov/sites/146/files/DelBay%20Annex%20F%20Response%20Tools.pdf)

### Inferences
- **How accident data feeds the model.** The model needs a causation probability Pc, the probability that a ship on a collision course fails to avoid the bridge.
  - Use a Norwegian prior: groundings and allisions per passage on comparable Norwegian fjord legs. Calculate it as Sdir grounding/allision counts divided by AIS-derived passages through Kystdatahuset passage lines.
  - Use the AASHTO and IALA defaults as cross-checks. AASHTO's base aberrancy rate is on the order of 0.6×10⁻⁴ for ships, with correction factors (background knowledge, not verified this session).
  - Use **MISLE allisions** with MarineCadastre transit counts to fit a US ship-bridge allision rate per transit, which validates the method on US bridges.
  - Use **blackout and drift frequency** for category II (loss of propulsion or steering) from MISLE "loss of propulsion" casualties, or from the literature. The NTSB Dali findings add a vivid example: two blackouts and a 58 s breaker delay (section 7).
- **NOR VTS incidents** carry wind fields and ship types, which make them a cheap source for conditional factors (wind and ship-type dependence) and for validating "near-miss" detection from AIS.
- **DAMA**, as listed in the brief, is the Norwegian Sdir legacy database (1981–2006), now merged. Researchers usually get extracts by agreement with Sdir. NTNU and SINTEF have used it in grounding and collision studies (background knowledge).
- Ship–bridge allisions are rare in Norway. Expect to rely on **Scandinavian and European pooled data**:
  - Sweden: Tjörnbron 1980, and Almöbron (background knowledge)
  - Denmark: Great Belt strikes
  - EMCIP aggregates and MISLE
  
  Treat these through a Bayesian update rather than a Norway-only frequency.

### Gaps
- No public Sdir bulk download found. I could not confirm the access procedure, fields (for example whether "kontaktskade/allisjon mot bru" is a coded category) or licence. Ask Sdir directly.
- **No source found for Norwegian ship–bridge strike counts for 2018–2026.** A targeted Norwegian news or NSIA search ("skip traff bru", "kontaktskade bru") could not be run because the search budget was exhausted.
- I did not check EMCIP's public-access portal and its allision/contact breakdown.
- I did not confirm whether MISLE extracts after 2015 are public.

---

## 6. Traffic-context datasets (pilotage, VTS, fairways, port and cruise calls, fish farms)

### Takeaway
Kystverket's published statistics give regional context:
- Passages on about 30 lines.
- Distance sailed by 13 ship types × 5 size classes since 2011, quality-checked in **Havbase**.

Fish-farm locations and polygons are fully open from **Fiskeridirektoratet** (NLOD; WFS, REST API, Shapefile/GeoJSON).

Pilotage, VTS and fairway layers exist in Kystverket's Kystinfo/Geonorge ecosystem, but I could not verify their exact endpoints or terms this session. Cruise-call calendars for Nordfjord ports were also not verified.

### Cited Findings
- **Passage statistics**: about 30 predefined AIS passage lines; annual counts by direction, ship category and size; ships with AIS off are missed — [Kystverket metadata: skipspasseringer](https://www.kystverket.no/contentassets/1faf2c86b6104e5e9ab5b4dfb0b289a6/metadata-om-statistikk-for-skipspasseringer.pdf/download)
- **Distance sailed**: time series for 13 ship types and five size categories in the North Sea, Norwegian Sea and Barents Sea, including waters inside the baseline, from 2011 — [Kystverket metadata: utseilt distanse](https://kystverket.no/contentassets/952946ba166948f88f57132b42986150/metadata-om-statistikk-for-utseilt-distanse.pdf/download)
- **Havbase** is Kystverket's system used to quality-control these statistics. It joins AIS with ship-register technical data. Data quality improved from 2013, so compare earlier series with care — [Kystverket metadata: driftsutslipp](https://www.kystverket.no/contentassets/456c82ae26df47d9b892c610e72f8493/metadata-om-statistikk-for-driftsutslipp-til-luft.pdf/download)
- **Kystdatahuset custom passage lines** let you count all passages across any user-drawn line — [COWI 2019](https://vestfoldfylke.no/globalassets/bypakke-tonsberg-regionen/dokumenter/overordnet-styringsgruppe-dokumenter/2019/21-november/sak_27-19_vedlegg_rap-kon-001-vedlegg-2-skipstrafikkanalyse.pdf)
- **Fiskeridirektoratet aquaculture**:
  - Locality midpoints and polygons, built from approved outer points, with status, species, production type, capacity and purpose.
  - Download portal `portal.fiskeridir.no/nedlasting` (Shapefile). WFS `FiskeridirWFS_akva` on `gis.fiskeridir.no`, plus a WMS "Akvakultur lokaliteter".
  - JSON API `https://api.fiskeridir.no/pub-aqua/`, and `https://api.fiskeridir.no/aqua-operations-public`.
  - Formats GML, SOSI, KML, GeoJSON, Shape and FGDB. CRS EPSG:25832/25833/25835/3035/4258.
  - **NLOD**, crediting Fiskeridirektoratet.
  
  Source: [Geonorge Aquaculture locations](https://www.geonorge.no/geonetworktest/srv/api/records/4ca8af5e-ffc7-4636-847d-4eca92c4a3b0); [INSPIRE record](https://inspire-geoportal.ec.europa.eu/srv/api/records/4ca8af5e-ffc7-4636-847d-4eca92c4a3b0) (search summary; the API catalogue entry was last updated 2026-03-20)
- The **BarentsWatch Fishhealth API** offers localities (`/bwapi/v1/geodata/fishhealth/localities/{year}/{week}`), vessel visits per locality (`/locality/{no}/vessel/{year}`) and vessel tracks per MMSI and week. BarentsWatch defines a "visit" as being within 400 m of the locality midpoint at under 1 knot — [bronnbat-ais source and README](https://github.com/mariu1914/bronnbat-ais)

### Inferences
(background knowledge, not verified this session, unless cited above)
- **Pilotage (los).**
  - Compulsory pilotage under the Pilotage Regulations (losforskriften) applies to vessels above set length thresholds, roughly 70 m, and to smaller passenger vessels and vessels carrying dangerous or polluting cargo.
  - Pilot exemption certificates (farledsbevis) are common for regular ferries and coastal traffic.
  - Kystverket's pilot-order system (Kystverket "los" / SafeSeaNet Norway) holds pilot-order data that a research application might obtain.
  - In the model: pilot-on-board reduces Pc, typically by a factor of about 0.3–0.5 in IALA IWRAP-style analyses. So record pilot status per transit: infer it from vessel size and type plus the ships' pilot exemption status, or obtain it from Kystverket.
- **VTS.** Norway's VTS centres (Vardø, Brønnøy, Kvitsøy, Fedje, Horten, Grenland/Brevik, among others) cover specific approaches. **Inner Nordfjorden is probably outside VTS coverage**, which matters because VTS presence is a Pc risk-reducer. VTS area polygons are published in Kystinfo/Geonorge. Verify both points.
- **Fairways (farleder).** Kystverket publishes a national fairway dataset ("Farleder": hovedled/biled, maybe with navigational-installation datasets) through Geonorge and Kystinfo WMS/WFS. Use it to define nominal routes and Pedersen category I lateral distributions where AIS is sparse, and to place aids to navigation in the 3D twin.
- **Cruise calls.** Cruise ships to Olden, Loen, Stryn and Nordfjordeid must pass the Anda–Lote section. Port call calendars are published by the Nordfjord port authority (Nordfjord Havn); verify the URL and format. Cross-check calendars against AIS-detected cruise passages, and use future calendars to project traffic growth.
- **Fish farms.** Fish farms near the crossing constrain manoeuvring room and add well-boat and service-vessel traffic, often small craft below the open-AIS 45 m leisure cut-off but included if fishing ≥15 m. Use the Fiskeridirektoratet polygons as obstacles in drift and grounding geometry and as 3D-scene objects.

### Gaps
- I did not verify Kystverket datasets and URLs for pilotage areas (lospliktige farvann), VTS polygons, fairways or Kystinfo services.
- I did not verify Havbase's public access (havbase.no innsyn).
- I found no Nordfjord Havn cruise calendar URL or number of annual cruise calls.
- I could not check whether the Fiskeridirektoratet locality API responds without a key (blocked domain).

---

## 7. Francis Scott Key Bridge (Baltimore, 26 March 2024): data for a hindcast and validation demo

### Takeaway
A credible Key Bridge hindcast can be built from four sources:
- **MarineCadastre 2024 GeoParquet** (CC0, 1-min) for ambient traffic and the Dali's approach.
- The **NTSB docket for DCA24MM031**: VDR and other electronic data factual report, plus more.
- The **NTSB final report** (18 Nov 2025, 259 pages) for the event timeline.
- **NTSB MIR-25-10** (March 2025) for the AASHTO Method II vulnerability inputs. NTSB computed the Key Bridge's risk at nearly **30× the acceptable threshold** for critical bridges.

BlueTopo provides the CC0 bathymetry.

### Cited Findings
- **NTSB MIR-25-10**, "Safeguarding Bridges from Vessel Strikes: Need for Vulnerability Assessment and Risk Reduction Strategies" (March 2025):
  - NTSB applied **AASHTO Method II** and found the Key Bridge was nearly **30 times above** the acceptable risk threshold for critical or essential bridges when the Dali struck it.
  - The Maryland Transportation Authority never ran the vulnerability assessment recommended by the 1991 and 2009 AASHTO specifications.
  - The report recommends assessments for **68 other bridges in 19 states**.
  
  Source: [NTSB MIR-25-10 PDF](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2510.pdf); [ENR](https://www.enr.com/articles/60481-ntsb-says-key-bridge-was-vulnerable-to-collapse); [Waterways Journal 2025-03-28](https://www.waterwaysjournal.net/2025/03/28/ntsb-recommends-bridge-collapse-risk-assessments/)
- **NTSB final report** released at a board meeting on **18 November 2025** (259 pages):
  - Probable cause: a **loose signal wire at a terminal block**, caused by improper installation of wire-label banding, led to loss of electrical power, propulsion and steering close to the bridge.
  - A second blackout came from low fuel pressure: a flushing pump was being used to feed the generators.
  - The high-voltage breakers were in **manual** mode. Automatic mode would have cut the initial blackout from **58 s to about 10 s**.
  - Recommendations went to USCG, FHWA and AASHTO.
  
  Source: [KATV/Sinclair syndicated report](https://katv.com/news/nation-world/ntsp-dali-key-bridge-collaspe-loose-wire-final-report); [Dundalk Eagle](https://www.dundalkeagle.com/key_bridge/ntsb-issues-final-report-detailing-cause-of-key-bridge-collapse/article_b91352cf-1d0e-4ad3-914a-a5232a7ddb36.html); [KRDO 2025-11-19](https://krdo.com/news/2025/11/19/how-the-ntsb-is-trying-to-prevent-another-major-bridge-collapse/)
- **VDR data.** The Dali's VDR was a JRC JCY-1900. USCG recovered about 6 hours of VDR data (00:00–06:00, 26 March 2024). Recorded sensor parameters were limited: speed, engine RPM, rudder angle, heading and some alarms. A "Voyage Data Recorder (VDR) and Other Electronic Data Group Chairman's Factual Report" was to be released in the public docket — [Seatrade Maritime](https://www.seatrade-maritime.com/casualty/what-dalis-black-box-recorder-tells-us-about-baltimore-bridge-allision); [copy of NTSB VDR audio report](https://img-oss.54seaman.com/Editor/Picture/20250604/7726079d4dc786094cc2745c05a792a2.pdf)
- NTSB criticised the VDR ecosystem:
  - data went unrecorded during the first underway blackout
  - bridge-to-engine-room communications were not recorded
  - the playback software was inadequate
  - the full data set could not be downloaded aboard
  - proprietary formats made the workflow complex
  
  Source: [KRDO 2025-11-19](https://krdo.com/news/2025/11/19/how-the-ntsb-is-trying-to-prevent-another-major-bridge-collapse/)
- **AIS**: `ais-2024-03-26.parquet` exists in the MarineCadastre 2024 daily GeoParquet set (CC0, 1-minute) — [daily file list](https://github.com/ocm-marinecadastre/ais-vessel-traffic/blob/main/data/daily-2024-parquet-filenames.md). Annual 2020–2024 transit-count services give the Patapsco/Fort McHenry channel traffic density for the AASHTO *N* term — [MarineCadastre repo](https://github.com/ocm-marinecadastre/ais-vessel-traffic)
- **Bathymetry**: BlueTopo/NBS tiles through `fetch_tiles(..., geometry='-76.6,39.1,-76.4,39.3')`-style bounding boxes, CC0 — [BlueTopo repo](https://github.com/noaa-ocs-hydrography/BlueTopo)

### Inferences
- **Hindcast demo pipeline.**
  - (1) Ambient traffic for 2023–2024 from MarineCadastre gives *N*, the lateral distribution and the speed/size mix for AASHTO Method II.
  - (2) Compare our Method II result with NTSB's "≈30× threshold" finding as a validation target (the AASHTO threshold for critical bridges is AF ≤ 1×10⁻⁴ per year; background knowledge).
  - (3) Rebuild the Dali's path from 1-min AIS, densified with the docket VDR or electronic-data factual report (heading, speed, rudder, RPM) and any docket AIS exports at native rate.
  - (4) Drive the 3D replay in Cesium with BlueTopo bathymetry and a parametric model of the bridge, its piers and its protection. Then run the drift-after-blackout module and check that the impact point and timing match.
- **Fetching the docket.** The NTSB docket is reached through NTSB CAROL/docket search with NTSB number **DCA24MM031**. The docket URL pattern `data.ntsb.gov/Docket/?NTSBNumber=DCA24MM031` is background knowledge, not verified this session. Docket items are US government works and generally public domain, but check for third-party copyrighted attachments.
- **1-minute AIS is too coarse for the final ~4–5 minutes.** Native-rate AIS for the Dali may be in the docket. Alternatively, raw NAIS can be requested from USCG NAVCEN (background knowledge), or bought from commercial providers (Spire or Kpler archives at native resolution).
- Vessel particulars for the Dali (LOA, beam, DWT, draught, displacement), the speed at impact and the pier-protection geometry should be taken from the NTSB final report rather than press reports; they were not confirmed this session.

### Gaps
- The docket contents list (whether native-rate AIS, ECDIS and VDR CSV exports are included) was not confirmed, because ntsb.gov was blocked and no search result described docket items.
- NTSB MIR-25-10's numeric Method II inputs (*N*, PA, PG, PC, PF and the computed AF) were not extracted, because the PDF could not be fetched.
- The Dali's speed at impact and its particulars were not confirmed from a primary source this session.

---

### Cross-cutting summary: dataset → model component mapping
(Inference; for the report writer)

| Model / scene component | Norway (Nordfjorden) source | US (Key Bridge) source | Licence / access risk |
|---|---|---|---|
| Transit counts *N*, lateral and speed distributions | Kystdatahuset passage line; HAIS GeoParquet; BarentsWatch (14 d only) | MarineCadastre 2024 GeoParquet / AccessAIS; transit-count services | NLOD open tier excludes small craft; full tier by application; US CC0 |
| Ship mass, size, air draft, bow | Kystdatahuset `ship/combined`; Equasis; Sdir register; regressions | Equasis; NTSB report (Dali) | Equasis no bulk use; Sea-web paid |
| Causation Pc and conditional factors | Sdir accident DB (by request); NOR VTS incidents (Kystdatahuset API); SSB | MISLE researcher files; NTSB reports | Sdir access unclear |
| Drift and blackout scenarios | NORA3 wind, NorKyst-800/v3 currents, vannstand API tide | NTSB VDR/electronic data; NOAA tides (background) | MET/Kartverket open (NLOD / CC BY 4.0) |
| Visibility / fog | ENSD METAR (Sandane at Anda); Frost API | NOAA/ASOS (background) | Free client ID |
| Terrain mesh | Høydedata DTM1/DTM10 | USGS 3DEP (background) | CC BY 4.0 / public domain |
| Seabed | Kartverket 50 m grid (finer classified inside 12 nm) | BlueTopo / CUDEM | CC BY 4.0 / CC0; ask sjodata@kartverket.no |
| Charts / fairways | Generalised depth contours (open S-57); ENC needs agreement; Kystverket fairways (unverified) | NOAA ENC (background) | Kartverket ENC restricted |
| Obstacles / context | Fiskeridirektoratet aquaculture WFS/API | — | NLOD |
| Regional density context | EMODnet vessel density 1 km | MarineCadastre transit counts | EMODnet open (licence unverified) |
| Method precedents | Bjørnafjorden (NTNU/SINTEF: ship impact dominates floating-bridge accidental hazards — [Moan, NTNU](https://folk.ntnu.no/jochenk/JCSS/Files/PRES_Moan.pdf); [SINTEF chapter](https://www.sintef.no/en/publications/publication/019cb30a396b-991386fa-497b-452f-8768-0498fc45cada/)); Chalmers AIS + simulator allision thesis ([Chalmers](https://research.chalmers.se/en/publication/521842)); AIS + Monte Carlo + Bayesian network ([PMC9025040](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9025040/)); passing-ship distribution sensitivity ([JMSE 2024 via DOAJ](https://doaj.org/article/70d119c1f64c4c19b810ed0c7cc6e768)) | NTSB MIR-25-10 | — |

Further reading not opened this session: Kress & Mitchell (2023), *AIS data: an overview of free sources*, ERDC/CHL CHETN-IX-62 ([CMTS PDF](https://www.cmts.gov/Portals/75/Documents/page_maritime_data/2023_Kress_Mitchell_AIS%20data%20an%20overview%20of%20free%20sources_ERDC-CHL_CHETN-IX-62.pdf)).
