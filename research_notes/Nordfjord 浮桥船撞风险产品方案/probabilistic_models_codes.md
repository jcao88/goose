# Probabilistic ship–bridge allision models, mitigation quantification and design codes (with Norwegian floating-bridge practice)

Research-method note for the report writer:
- WebFetch failed with DNS errors on every host. Direct curl downloads were refused by the session's egress policy (HTTP 403) for iala.int, chalmers.se, diva-portal.org, sciencedirect.com, ntsb.gov, vegvesen.no, pianc.org, infrasweden.nu and others. Per the proxy rules these were not worked around.
- The shared web-search budget then ran out partway through the work (limit 200 per turn across all agents), so the planned searches on VTS/TSS/pilotage reduction values, Bayesian-network causation models, Halsafjorden/Sulafjorden and Nordfjorden theses were **not** run.
- Findings below therefore come from three places:
  - (a) extended-search result summaries (snippet level; flagged where the attribution is uncertain);
  - (b) the full source of **OMRAT**, an open-source (GPL) IWRAP-equivalent tool by Axel Hörteborn. It was cloned from GitHub (commit d4022ecc, 15 Oct 2026) and its code and theory docs were read directly;
  - (c) small numerical checks I ran myself, labelled as such.
- Treat every value here that was not read in a primary document as "verify before use".

## Q1. IWRAP Mk2 modelling principles: powered (Cat I/II) and drifting allision, default parameters

### Takeaway
IWRAP Mk2 computes every accident frequency as F = N_A·P_C:
- N_A is a geometric "candidate" count from traffic volume and a lateral-distribution mixture (up to 3 normal plus 1 uniform component).
- P_C is a causation factor (IALA defaults ≈1–2×10⁻⁴ for powered accidents; 1 for drifting).
- Powered allision has two categories. Category I covers ships already in the lane: Q·P_C·∫f(z)dz over the obstacle. Category II covers failure to turn at a bend, adding a Poisson "no position check" decay exp(−d/(λV)) with λ≈180 s.
- Drifting allision = (exposure hours × blackout rate) × wind-rose direction share × geometric "probability hole" × P(not repaired before reaching the obstacle), with an anchoring shadow ahead of the obstacle.

The open-source OMRAT code reproduces this structure and documents all of its defaults, which makes it the most practical blueprint for an open engine.

### Cited Findings

**Core framework and equations**
- General form: every IWRAP accident frequency is F_accident = N_A·P_C. N_A is the geometric candidate count from geometry and traffic alone; P_C is the conditional probability that a candidate becomes an accident. — [OMRAT theory.rst](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst)
- **Category I** (ships on the ordinary route whose lane overlaps the obstacle): N_I = P_c·Q·∫_{z_min}^{z_max} f(z)dz. For a normal lane this is Φ((z_max−μ)/σ) − Φ((z_min−μ)/σ). OMRAT cites this as "Hansen Eq. 4.15". — [OMRAT basic_equations.py](https://github.com/axelande/OMRAT/blob/main/compute/basic_equations.py)
- **Category II** (fail to turn at a bend): N_II = P_C·Q·f(z)·exp(−d/a_i), with a_i = λ·V.
  - d is the distance from the bend to the obstacle, λ the mean time between position checks, V the ship speed.
  - OMRAT cites this as "Hansen Eq. 4.16–4.17".
  - Source: [OMRAT powered.rst](https://github.com/axelande/OMRAT/blob/main/help/source/powered.rst)
- The IALA wiki states that position checking is modelled as a Poisson process, with λ ≈ 3 minutes (180 s). — [IALA IWRAP wiki, Predicting Grounding Frequencies](https://www.iala.int/wiki/iwrap/index.php/Predicting_Grounding_Frequencies) (via search snippet)
- The same grounding causation factor applies to both Category I and II in IWRAP. The Category II formula is exact only when the grounding line is orthogonal to the route, which is rarely the case. — [IALA IWRAP wiki](https://www.iala.int/wiki/iwrap/index.php/Predicting_Grounding_Frequencies); [IWRAP theory doc](https://www.iala.int/wiki/iwrap/images/2/2b/IWRAP_Theory.pdf) (search snippets)
- Sensitivity to λ (snippet level; attribution to be verified): one study found that Pedersen's approach gives more grounding candidates than the actual traffic (893 tankers) for mean check times above 100 s. A worked case at a = 180 s and V = 11.78 kn gave a grounding probability of ≈0.0456. — search result set incl. [Ocean Eng. "Grounding risk quantification in channel using the empirical ship domain"](https://www.sciencedirect.com/science/article/abs/pii/S0029801823020565)

**Lateral distribution**
- Each leg and direction uses a mixture: f(z) = Σ_{i=1..3} w_i·φ((z−μ_i)/σ_i) + w_u·U(z;a,b), with weights summing to 1, fitted from AIS or set by hand. — [OMRAT theory.rst](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst)
- OMRAT samples the lateral distribution with 1000 rays over μ±6σ. The span was 4σ with 500 rays before 2 Oct 2026.
  - The reason for the change: normal mass beyond 4σ is 6.3×10⁻⁵ per side. On a 10,000 ships/yr leg that is an accident frequency of order 10⁻⁴/yr whenever an obstacle sits in that tail.
  - Source: [OMRAT powered.rst](https://github.com/axelande/OMRAT/blob/main/help/source/powered.rst)
- OMRAT added a "Category I start zone" fix, default on, which ignores hits in the first 10 % of a leg. Near the origin waypoint, ships are still arriving from the previous leg, and the fitted distribution's far tail produced spurious hits (ESCOW wind-farm case, 1 Oct 2026). — [OMRAT cat1_start_zone.py](https://github.com/axelande/OMRAT/blob/main/compute/cat1_start_zone.py)

**Default causation factors**
- Head-on 0.5×10⁻⁴; overtaking 1.1×10⁻⁴; crossing 1.3×10⁻⁴; bend 1.3×10⁻⁴; merging 1.3×10⁻⁴; powered grounding ("forgetting to turn") 1.6×10⁻⁴. — [Mombasa study, J. Positioning Navigation Timing](https://www.glonav.org/m/journal/view.php?number=83); [KOSOMES](http://www.kosomes.or.kr/journal/article.php?code=19817) (search snippets)
- The MONALISA 2.0 report says these defaults are mainly rooted in observations by Fujii & Mizuki (1998). — [MONALISA 2.0 D4.3.2](https://s3-eu-west-1.amazonaws.com/stm-stmvalidation/uploads/20160420145707/ML2-D4.3.2-Collision-and-conflict-candidates-causation-factors.pdf) (snippet)
- OMRAT's IALA default table:

  | Accident type | Default P_C |
  |---|---|
  | Head-on | 4.9×10⁻⁵ |
  | Overtaking | 1.1×10⁻⁴ |
  | Crossing | 1.3×10⁻⁴ (Pedersen) |
  | Bend | 1.3×10⁻⁴ |
  | Powered grounding | 1.6×10⁻⁴ |
  | Allision (structure) | 1.9×10⁻⁴ (Fujii et al. 1974) |
  | Drifting | 1.0 (powerless ship) |

  Merging defaults to the crossing value, "IWRAP does not publish a separate figure". — [OMRAT theory.rst](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst); [OMRAT iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py); [OMRAT collisions.rst](https://github.com/axelande/OMRAT/blob/main/help/source/collisions.rst)
- **Conflict on the allision/grounding default.** OMRAT's IWRAP-XML exporter falls back to different values from its own defaults:
  - p_allision_causation = p_allision_no_turn_causation = p_grounding_causation = p_grounding_no_turn_causation = 0.000155 (1.55×10⁻⁴);
  - head-on 5e-05, overtaking 0.00011, crossing/bend/merging 0.00013, drifting 1.
  - This suggests IWRAP files carry 1.55×10⁻⁴, not 1.6 or 1.9×10⁻⁴, for powered grounding/allision. Verify against an actual IWRAP installation.
  - Source: [OMRAT iwrap_convertion.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_convertion.py)
- Older values cited by IALA (MacDuff): 10⁻⁴ for crossings and 5×10⁻⁴ for head-on. — search summary of IALA material ([IWRAP Mk2 manual v5.3](https://www.iala.int/wiki/iwrap/images/7/75/IWRAP_Mk2_Manual_v5.3.pdf))
- The IWRAP manual says the defaults were selected by IALA experts. Altering them is "not recommended"; users should instead apply causation **reduction factors**, dividing P_C by the RF. Ferry types have a default RF of 20. — [IWRAP Mk2 Manual v5.3 (GateHouse, Engberg 2017)](https://www.iala.int/wiki/iwrap/images/7/75/IWRAP_Mk2_Manual_v5.3.pdf) (snippet)
- IWRAP XML global settings include `fastferry_reduction_factor = 20` and `passengership_reductionfactor = 20`.
  - Per-leg reduction factors also exist: `allision_causation_rf`, `allision_no_turn_rf`, `allision_drifting_rf`, `grounding_*_rf`, `headon_causation_rf`, `overtaking_causation_rf` and `aton_reduction_factor`, all default 1. A per-ship-type `causation_reduction_factor` also exists.
  - OMRAT writes `meantime_between_checks = 240` when unset. The unit is presumably seconds; this conflicts with the wiki's 180 s.
  - Source: [OMRAT iwrap_convertion.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_convertion.py)

**Collision geometric candidates (for completeness)**
- Head-on and overtaking: N_G = (Q₁/V₁)(Q₂/V₂)·V_ij·P_G·L_w/(s/yr), where P_G = Φ((μ_ij+B_ij)/σ_ij) − Φ((μ_ij−B_ij)/σ_ij).
- Crossing: N_G = (Q₁/V₁)(Q₂/V₂)·V_ij·D_ij/|sin θ|, with V_ij = √(V₁²+V₂²−2V₁V₂cos θ). D_ij is the Minkowski collision diameter.
- Source: [OMRAT collisions.rst](https://github.com/axelande/OMRAT/blob/main/help/source/collisions.rst)

**Drifting allision**
- Most ships experience about one main-engine blackout per year, typically 0.1–2 per year. Blackouts along a leg segment are modelled as a Poisson process. — [IALA IWRAP wiki](https://www.iala.int/wiki/iwrap/index.php/Predicting_Grounding_Frequencies) (snippet)
- The IWRAP theory document gives blackout rates of 0.1 /yr for Passenger/Ro-Ro and 0.75 /yr for other vessels. — [IWRAP Theory (Friis-Hansen)](https://www.iala.int/wiki/iwrap/images/2/2b/IWRAP_Theory.pdf) (snippet)
- OMRAT's defaults conflict with the theory document's 0.75: drift_p = 1.0 /ship-yr, overridden to 0.1 for passenger/ferry/ro-pax/ro-ro/cruise. — [OMRAT basic_equations.py](https://github.com/axelande/OMRAT/blob/main/compute/basic_equations.py)
- **Repair time (default Weibull).** The IWRAP wiki gives scale a = 1.05 and shape b = 0.9, giving a mean of 1 h and a standard deviation of 1.13 h. The IALA 2017 training deck uses the same pair. — [IALA IWRAP wiki](https://www.iala.int/wiki/iwrap/index.php/Predicting_Grounding_Frequencies); [IWRAP Mk2 training, Engberg 2017](https://www.iala.int/wiki/iwrap/images/5/51/IWRAP_Mk2_GH_2017.pdf) (snippets)
- The v4.7.0 manual states that the standard repair-time Weibull parameters were adjusted. The previous settings came from old literature; the new values came from comparison with the Dutch **SAMSON** project. The new numbers were not retrieved. — [IWRAP Mk2 v4.7.0 Manual](https://www.iala.int/wiki/iwrap/images/d/d0/Iwrapmk2.pdf) (snippet)
- **Repair time (OMRAT default).** OMRAT's labelled "IWRAP-compatible" repair default is a **lognormal** with s = 0.95, loc = 0.2 h, scale = 0.85 h. Its worked examples use lognormal(1, 0, 1), which leaves ≈11.8 % of blackouts unrepaired at 3.28 h. — [OMRAT iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py); [OMRAT drifting.rst](https://github.com/axelande/OMRAT/blob/main/help/source/drifting.rst)
- **Anchoring.** IWRAP lets the user specify an anchoring probability, but a minimum distance is required for the ship to anchor. v5.3.0 added anchor handling to drifting. — [IWRAP v4.7](https://www.iala.int/wiki/iwrap/images/d/d0/Iwrapmk2.pdf), [v5.3 manuals](https://www.iala.int/wiki/iwrap/images/7/75/IWRAP_Mk2_Manual_v5.3.pdf) (snippets)
- OMRAT anchoring defaults: anchor_p = 0.70; anchoring is possible where depth < 7 × draught. — [OMRAT iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py); [OMRAT user guide](https://axelande.github.io/OMRAT/user_guide.html)
- **Drift direction and speed.** OMRAT defaults to a uniform 8-direction wind rose (0.125 each), editable from wind statistics, and a 1.0 kn drift speed. — [OMRAT iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py)
- **Drifting equation set (OMRAT):**
  - hours_present = L_leg/(V·1852)·f_ships/yr
  - base = hours_present·λ_bo/(365.25·24)
  - P = base·r_p,d·h_X,eff·P_NR(d_X), where P_NR(d) = 1 − F_repair(d/(V_drift·3600))
  - Obstacles are processed nearest-first in a cascade. Anchoring removes a share: remaining *= (1 − a_p·h_anchor). Allision/grounding obstacles shadow those behind them: remaining *= (1 − h).
  - Structures count only if structure height < ship height.
  - Reach distance = V_drift·3600·t₉₉.
  - The probability hole h is a pure geometric double integral (s along the leg × lateral y over ±5σ) of rays cast in the drift direction.
  - Source: [OMRAT computation_procedures.md](https://github.com/axelande/OMRAT/blob/main/compute/computation_procedures.md)
- **IWRAP-equivalence verification.** Hörteborn verified IWRAP Mk II allision modelling against OMRAT, which is built on the same theory. He found IWRAP uses the same equations for allisions and groundings; some parameters act linearly while others interact with model layout. — [Hörteborn, ICCGS 2023 (CRC 2024, pp. 51–58)](https://www.taylorfrancis.com/chapters/edit/10.1201/9781003462170-9/allision-modelling-iwrap-mk-ii-verification-sensitivity-study-axel-h%C3%B6rteborn); [RISE DiVA record](https://ri.diva-portal.org/smash/record.jsf?pid=diva2:1854644&dswid=3733)
- **OMRAT provenance and licence.** "Open Maritime Risk Analysis Tool", © 2022–2026 Axel Hörteborn and contributors, a QGIS plugin.
  - The README says GPL v2+, but metadata.txt says GPL-3; this needs checking before productisation.
  - A stated roadmap goal is to "make the calculation IWRAP-bit-identical for cross-validation".
  - It has bidirectional IWRAP XML import/export.
  - Source: [OMRAT repo README/LICENSE](https://github.com/axelande/OMRAT)
- **Pedersen (1995)** "Collision and Grounding Mechanics", WEMT'95, pp. 125–157, defines four track-pattern categories:
  - I: ordinary ships on the route offset;
  - II: failure to turn;
  - III: evasive manoeuvres;
  - IV: other, e.g. drifting.
  - It is used for the Great Belt and Øresund analyses, together with Hauge et al. 1998. — [Hörteborn, PIANC 2025](https://www.pianc.org/wp-content/uploads/2025/05/Axel-Horteborn.pdf); [Pedersen 2002, Proc IMechE M](https://journals.sagepub.com/doi/10.1243/147509002320382121); [Pioneer of Ship Collision and Grounding (Ships & Offshore Struct.)](https://www.tandfonline.com/doi/full/10.1080/17445302.2020.1855499)
  - Pedersen et al. (2020) later modified the fourth scenario. — [Hörteborn & Eidem, J. Navigation 2025](https://www.researchgate.net/publication/394412411_Probability_of_active_navigational_failures_incident_analyses_for_use_in_ship-bridge_allision_risk_assessments)

### Inferences
- **Weibull parameterisation (my own calculation).** The wiki's "a = 1.05, b = 0.9 → mean 1 h, sd 1.13 h" matches F(t) = 1 − exp(−a·t^b) (mean 0.997 h, sd 1.11 h). It does not match the scale form 1 − exp(−(t/a)^b), which gives mean 1.105 h and sd 1.23 h. An implementer should use the rate form to reproduce IWRAP.
- **Tail difference between repair defaults (my own calculation).** P(not repaired):

  | Time | IWRAP Weibull (1.05, 0.9) | OMRAT lognormal (0.95, 0.2, 0.85) |
  |---|---|---|
  | 1 h | 0.35 | 0.53 |
  | 2 h | 0.14 | 0.22 |
  | 3 h | 0.06 | 0.11 |
  | 6 h | 0.005 | 0.022 |

  The lognormal default yields roughly 2–4× higher drifting allision for obstacles 2–6 h of drift away. Repair-time choice is a first-order driver for fjord crossings where the bridge is 0.5–3 NM from the lane.
- **Category II scale (my own calculation).** At V = 10 kn and λ = 180 s, a_i ≈ 926 m. exp(−d/a) = 0.58, 0.34, 0.12 and 0.005 at d = 0.5, 1, 2 and 5 km. Missed-turn allision is therefore dominated by bends within ~2 km of the bridge, which is typical for fjord approaches.
- **Engine design.** An open IWRAP-equivalent engine can be specified almost line-by-line from OMRAT's documented equations and defaults. Its known divergences from IWRAP are listed above and form a ready-made test matrix:
  - Cat I start-zone trimming;
  - 6σ ray span;
  - lognormal repair default;
  - 1.0 versus 0.75 blackout rate;
  - 1.9×10⁻⁴ versus 1.55×10⁻⁴ allision P_C;
  - 240 s versus 180 s check time.

### Gaps
- I could not read the primary IWRAP theory document (Friis-Hansen 2008) or the v5.3 manual directly. Exact equation numbering, the current (SAMSON-revised) Weibull repair parameters, IWRAP's default drift speed and anchoring probability, and the official mean-time-between-checks default remain unverified. The candidates are 180 s (wiki) and 240 s (OMRAT export fallback).
- The full IWRAP ship-type and size-class taxonomy (ship-type list, LOA bins) was not retrieved. OMRAT maps AIS types to IWRAP types (e.g. HSC → "Fast ferry"), but the IWRAP size bins were not confirmed.
- Fujii (1974) and Fujii & Mizuki (1998) were not read in the original.

## Q2. How mitigation measures (VTS, TSS, pilotage, tugs, speed limits) are quantified

### Takeaway
In IWRAP-type models, mitigation enters almost entirely as **divisors (reduction factors) on the causation factor**, plus geometric changes. Default reduction factors found:
- ferries/passenger ships ÷20 (IWRAP default);
- pilot on board ÷3 (COWIconsult);
- poor visibility ×2 for 3–10 % of the time and ×8 for 10–30 %.

Geometric changes include route or TSS layout, lateral distribution, speed (which changes a_i = λV and exposure time), and anchoring/emergency-tow probability in drifting. I could not verify quantitative VTS, TSS or tug-escort factors in this session.

### Cited Findings
- IWRAP recommends representing local conditions through causation **reduction factors** (P_C/RF) rather than editing the defaults. Ferry types default to RF = 20. — [IWRAP Mk2 Manual v5.3](https://www.iala.int/wiki/iwrap/images/7/75/IWRAP_Mk2_Manual_v5.3.pdf) (snippet)
- Typical local adjustment factors:
  - ferry/passenger vessels divide by 20 (two navigators, well-known route);
  - pilot on board divide by 3 (COWIconsult);
  - poor visibility 3–10 % multiply by 2;
  - poor visibility 10–30 % multiply by 8.
  - Source: [OMRAT theory.rst](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst)
- IWRAP exposes per-leg reduction factors by accident mechanism (all default 1), so a measure can be applied selectively:
  - powered Cat I `allision_causation_rf`;
  - Cat II `allision_no_turn_rf`;
  - `allision_drifting_rf`;
  - the equivalents for grounding;
  - `headon_causation_rf`, `overtaking_causation_rf`;
  - an `aton_reduction_factor` for aids to navigation.
  - It also has a per-ship-type `causation_reduction_factor`.
  - Source: [OMRAT iwrap_convertion.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_convertion.py)
- OMRAT applies user "risk-reduction factors" to drifting totals after the cascade (default 1.0; set below 1 to model, e.g., pilot on board). — [OMRAT reference_data_format.rst](https://github.com/axelande/OMRAT/blob/main/help/source/reference_data_format.rst)
- Askeland et al. (ESREL 2021, pp. 614–621) assessed allision risk for E39 extreme fjord crossings. The authors were Askeland, Dørum, Eidem, Randrup-Thomsen, Pedersen and Veie (NPRA/COWI/DTU authorship).
  - The analysis focused on the **effect of proposed risk-reducing measures** rather than absolute risk values.
  - A "Heinrich factor" is often used for such crossings but was not applied.
  - Impact analysis, i.e. how a floating bridge absorbs and dissipates energy, needs more attention.
  - Source: [DTU Orbit](https://orbit.dtu.dk/en/publications/risk-assessment-of-ship-allision-in-extreme-fjord-crossings/); [ResearchGate](https://www.researchgate.net/publication/354860829_Risk_Assessment_of_Ship_Allision_in_Extreme_Fjord_Crossings)
- STAPS-cons (Bjørnafjorden case): a 20 % longer duration of a ship missing its turning point notably raised both allision frequency and the energy the bridge must withstand. This implies that measures shortening detection/response time (bridge watch alarms, VTS intervention) act directly on the dominant parameter. — [Hörteborn et al., Engineering Structures 2025](https://www.sciencedirect.com/science/article/pii/S0141029625021224); [Chalmers thesis](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf)
- In AASHTO Method II, protective systems (fenders, dolphins, islands) enter as a protection factor PF in AF = N·PA·PG·PC·PF. Traffic and current conditions enter through PA correction factors (location, parallel current, cross-current, traffic density). — [FIU ABC-UTC AASHTO overview](https://abc-utc.fiu.edu/wp-content/uploads/2024/08/Vessel-Collision-Presentation.pdf); [JMSE 2024, 12(3):452](https://doi.org/10.3390/jmse12030452)

### Inferences
- **VTS, ship-collision warning systems and alarms.** In a Category II framework, a monitoring measure can be modelled two ways: as a reduced effective λ (more frequent "position checks", including external ones by VTS), or as a reduction factor on `allision_no_turn_rf`. The λ route is more physically grounded and is consistent with Hörteborn's finding that response duration drives risk.
- **Speed limits.** These reduce impact energy (∝v²) more than frequency. In Category II, slower ships have a smaller a_i = λV, so exp(−d/a_i) falls. In drifting, exposure hours on the leg actually increase as L/V.
- **Tug escort and emergency towing.** These are best represented in drifting as a modified repair/"rescue" time distribution (P_NR) or as an additional "arrest" probability analogous to anchoring.

### Gaps
- No verified quantitative values were obtained for VTS (e.g. % reduction in collision/grounding P_C), TSS, tug escort, AIS-based bridge warning or speed limits. The planned searches were blocked by the search-budget limit.
  - Unverified sources the team should consult: IALA Guideline 1018 (Risk Management) and the IWRAP wiki "Interpretation of causation factors in IWRAP" page; Danish Maritime Authority/COWI FSA studies; IMO FSA navigational-safety submissions; Friis-Hansen & Simonsen's GRACAT Bayesian network; Hänninen & Kujala (RESS 2012) on Gulf of Finland collision BBN.
  - Do not quote numbers for these from memory.
- The basis of the "pilot ÷3 (COWIconsult)" and "visibility ×2/×8" factors could not be traced past OMRAT's documentation.

## Q3. Critiques, validation and uncertainty of IWRAP-type models

### Takeaway
The main recent critiques come from Hörteborn and co-authors (Chalmers/RISE, with NPRA's Eidem):
- IWRAP Mk2 gave ≈4.5× higher allision probability than the mid-fidelity STAPS simulation in the base case (RESS 2025).
- Both models are very sensitive to failure duration and route layout.
- Incident data from 15 Scandinavian bridges show that the common assumptions (2 % of passages "errant", uniform hit probability along the bridge, reuse of the missed-turn causation factor) do not hold (J. Navigation 2025).

Default causation factors are judgement-based and mainly rooted in Fujii's Japanese strait data.

### Cited Findings
- **RESS 2025 (vol. 260, art. 111026), "Probabilistic analysis of ship-bridge allisions when designing bridges".**
  - It compares low-fidelity IWRAP Mk2 with the mid-fidelity STAPS (Monte Carlo, ship manoeuvrability and motion physics).
  - IWRAP Mk2 estimates ≈4.5× the allision probability of STAPS in the base case and "overestimates" accident probability.
  - Both models are sensitive to failure duration and route layout.
  - IWRAP is recommended for early design phases, STAPS for later ones.
  - IWRAP's drift assumes constant drift speed and ignores hydrodynamic characteristics.
  - The result rests on two case studies only.
  - Source: [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0951832025002273); [Trafikverket DiVA full text](https://trafikverket.diva-portal.org/smash/get/diva2:2051395/FULLTEXT01.pdf); [Chalmers record](https://research.chalmers.se/en/publication/545746)
- **Hörteborn & Eidem, J. Navigation 2025, "Probability of active navigational failures".**
  - Data: accident databases, incident reports and AIS for **15 bridges in Scandinavia over 6 years**.
  - Allision probability from active navigational errors varies widely between bridges.
  - The "currently common practice of 2 % uniform distribution of the number of ship passages on all bridges" is **not recommended**.
  - For "Wrong Course at a Turning point", strike probability is concentrated in a **cone beyond the last turning point**, not uniform along the bridge.
  - The paper tests the 2 % share, equal hit probability, and use of the missed-turn causation factor.
  - Citation conflict: the Cambridge page gives pp. 171–189 while the PDF header reads 1–19.
  - Source: [Cambridge Core](https://www.cambridge.org/core/journals/journal-of-navigation/article/probability-of-active-navigational-failures-incident-analyses-for-use-in-shipbridge-allision-risk-assessments/4457B860D16A6EF1C8E0B570E047A9EB); [ResearchGate](https://www.researchgate.net/publication/394412411_Probability_of_active_navigational_failures_incident_analyses_for_use_in_ship-bridge_allision_risk_assessments)
- **Hörteborn, PIANC 2025, "Probability for Ship-Bridge Allisions in the USA".** It applies the allision-probability framework to US bridges. Pedersen's 1995 categories are widely used as a standard, but critics note they do not explain *why* ships leave the route. — [PIANC 2025 paper](https://www.pianc.org/wp-content/uploads/2025/05/Axel-Horteborn.pdf)
- **STAPS (Ocean Engineering 2021).** It uses AIS data and a ship-manoeuvring simulator. Pedersen-type causation is estimated as accidents/allisions divided by ship passages, so accuracy depends on local accident statistics and traffic data. — [Hörteborn et al., Ocean Eng. 2021](https://www.sciencedirect.com/science/article/pii/S0029801821008088); [Licentiate thesis 2021](https://infrasweden.nu/wp-content/uploads/2024/07/Ship-Behaviour-and-Ship-Bridge-Allision-Analysis.-Licentiate-Thesis-2021.pdf)
- **Default causation factors.**
  - They are mainly rooted in Fujii & Mizuki (1998). — [MONALISA 2.0](https://s3-eu-west-1.amazonaws.com/stm-stmvalidation/uploads/20160420145707/ML2-D4.3.2-Collision-and-conflict-candidates-causation-factors.pdf) (snippet)
  - Some IWRAP application papers note that "there is currently no reliable method to fix the causation factor". — snippet from the result set; attribution uncertain, possibly [IOP J. Phys. Conf. Ser. 2311 012015](https://iopscience.iop.org/article/10.1088/1742-6596/2311/1/012015/pdf)
- AIS data quality is a recognised uncertainty source. Dempster-Shafer propagation of AIS-induced uncertainty has been proposed (Talavera et al., RESS 111, 2013, 95–105). — cited in [OMRAT theory.rst references](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst)
- **Tail sensitivity of the lateral distribution.** Truncating the normal at 4σ versus 6σ changes allision frequency by order 10⁻⁴/yr on a 10,000 ships/yr leg. That is the same order as the Norwegian acceptance limit. — [OMRAT powered.rst](https://github.com/axelande/OMRAT/blob/main/help/source/powered.rst)

### Inferences
- **Bias of the 4.5× factor.** It is a model-to-model ratio, not validation against accidents. Combined with the J. Navigation finding (strong bridge-to-bridge variability), it suggests IWRAP defaults are probably conservative for well-run Norwegian fjord traffic but cannot be assumed so at every site.
- **Calibration.** An open engine should allow per-site calibration of P_C, via a Bayesian update from local incident counts (Kystverket/DAMA, NPRA incident reports). It should also report results with an uncertainty band rather than a point value.
- **Consequences near 10⁻⁴.** Because the acceptance limit (10⁻⁴/yr) sits in the extreme tails, numerical choices (σ truncation, start-zone trimming, repair-time tail) can flip a pass/fail verdict. They must be fixed in a documented calculation standard.

### Gaps
- I could not read the "generalizing default causation factors" study (Chalmers/TUHH) mentioned in the brief, and found no reliable source for it in this session.
- Full numerical results of the RESS 2025 case studies (absolute frequencies, energy distributions) and of the PIANC US validation were not retrieved.
- No systematic validation of IWRAP allision predictions against Norwegian accident statistics was found.

## Q4. Design codes and acceptance criteria (N400, Eurocode EN 1991-1-7, AASHTO Method II, NTSB 2025) and derivation of the design ship/energy

### Takeaway
- **Norway (N400):** accidental actions with annual probability below 10⁻⁴ may be disregarded. Ship impact must be based on a project-specific risk analysis that fixes the design ship (size, mass, speed) and the design impact energy and momentum, so the risk acceptance criteria are met.
- **AASHTO Method II:** AF = N·PA·PG·PC(·PF), with AF < 10⁻⁴/yr for critical/essential bridges and < 10⁻³/yr for typical bridges.
- **NTSB (2025):** found the Key Bridge about 30× over the critical-bridge threshold and asked owners of 68 bridges to run Method II.
- **Eurocode EN 1991-1-7:** gives indicative ship classes and forces (Table C.4) and a dynamic force model, with risk analysis in an informative annex.

### Cited Findings
- **N400, 10⁻⁴ rule.** Accidental events with annual probability below 10⁻⁴ ("one event per 10,000 years") may be disregarded when the accidental load can be determined by probability calculations. An N400 table (2014 edition) states the same and refers to risk analysis for accidental load from ship traffic. — [Trøndelag fylkeskommune RoS analysis](https://www.trondelagfylke.no/contentassets/6ec84aec1dbf41a58a087a33eb40e0fb/ros-rap-001-ros-analyse.pdf); [N400 (2014) mirror](https://fileserver.motocross.io/trafikksiden/HB_N400_Bruprosjektering_2014.pdf); [N400 2009, NPRA Brage](https://vegvesen.brage.unit.no/vegvesen-xmlui/bitstream/handle/11250/283057/Hb-N400.pdf)
- **N400, risk analysis content.** Norwegian bridge rules require a separate risk analysis for the design ship that sets its size, mass and speed at collision, plus impact energy and momentum, so that the risk acceptance criteria are met. The analysis comprises frequency analysis and impact analysis. — [Hörteborn, "Risk analysis methodology for ship-bridge allisions" (Chalmers thesis)](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf)
- **N400, administrative details.** Design service life is 100 years. NPRA issued a 2021 hearing draft of a revised N400, and amendment circulars (e.g. NA-rundskriv 2017/09) exist, so the current edition must be checked. — [NPRA N400 page](https://www.vegvesen.no/fag/publikasjoner/handboker/vegnormalene/n400/); [N400 hearing draft 2021](https://www.vegvesen.no/globalassets/fag/horinger/2021/20-24090-56-forslag-til-revidert-vegnormal-n400-bruprosjektering-frist-20213006/forslag-til-revidert-vegnormal-n400-bruprosjektering.pdf)
- **N400 in practice.** A Norwegian county floating-bridge RoS computed annual ship-impact probability using **AASHTO methodology**, summing per-passage probabilities with PG dependent on available fairway width and ship dimensions. It explicitly did not assess consequence or collapse probability. — [Trøndelag fylkeskommune RoS](https://www.trondelagfylke.no/contentassets/6ec84aec1dbf41a58a087a33eb40e0fb/ros-rap-001-ros-analyse.pdf)
- **Eurocode EN 1991-1-7.**
  - Frontal static-equivalent design forces from seagoing vessels "should be specified", and Table C.4 (Annex C) gives an indicative classification of such ships. — [EN 1991-1-7:2006 text](https://www.phd.eng.br/wp-content/uploads/2015/12/en.1991.1.7.2006.pdf)
  - Annex C's dynamic model uses F = v_r·√(k·m), shown with experimental comparisons. — [JRC Eurocode training slides (Vrouwenvelder 2008)](https://eurocodes.jrc.ec.europa.eu/sites/default/files/2022-06/EN1991_7_Vrouwenvelder.pdf)
- **AASHTO Method II.**
  - AF = (N)(PA)(PG)(PC)(PF); PF was added in the 2024 AASHTO LRFD edition.
  - AF must be < 0.0001 for critical/essential bridges and < 0.001 for typical bridges, with the owner classifying the bridge.
  - Source: [FIU ABC-UTC presentation 2024](https://abc-utc.fiu.edu/wp-content/uploads/2024/08/Vessel-Collision-Presentation.pdf); [Bridge Engineering Handbook ch. 60](http://freeit.free.fr/Bridge%20Engineering%20HandBook/ch60.pdf)
- **AASHTO PA.** PA = BR·RB·RC·RXC·RD: base rate × corrections for bridge location, current parallel to the transit path, cross-current and traffic density. One paper takes BR = 1.2×10⁻⁴. — [JMSE 2024, 12(3):452](https://doi.org/10.3390/jmse12030452)
- **NTSB MIR-25-10 (2025).**
  - The Key Bridge's annual frequency of collapse was ≈**30×** AASHTO's acceptable level for critical bridges. MDTA never performed the risk calculation recommended by AASHTO 1991/2009.
  - NTSB asked owners (≈30 agencies) of **68 bridges in 19 states**, all built before 1996, to compute the **AASHTO Method II** annual frequency of collapse and report whether it exceeds the threshold.
  - NTSB also recommended an FHWA/USCG/USACE interdisciplinary team.
  - It stressed that the 68 bridges are not certain to collapse.
  - Source: [NTSB MIR2510](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2510.pdf); [ENR](https://www.enr.com/articles/60481-ntsb-says-key-bridge-was-vulnerable-to-collapse); [WorkBoat](https://www.workboat.com/coastal-inland-waterways/ntsb-says-68-u-s-bridges-should-be-assessed-for-risk-of-allision-collapse)
- **Frequent low-energy events.** A review of the floating-bridge risk procedure proposed scrutinising impact frequency for relatively frequent low-energy events, in addition to the 10⁻⁴ events relevant for the ALS criterion, and noted a relatively high estimated road-traffic disruption rate. — snippet; attribution likely [Review of PRA models for ship collisions with structures (ResearchGate 359602497)](https://www.researchgate.net/publication/359602497_Review_of_Probabilistic_Risk_Assessment_Models_for_Ship_Collisions_with_Structures)

### Inferences
- **Two ways to convert frequency to a design load.**
  - (i) **Exceedance-curve method:** sum frequencies over all ship classes, mechanisms (Cat I, Cat II, drifting) and impact scenarios. Build F(E > e) for kinetic energy E = ½(1 + c_a)·m·v², using impact speed, which differs between powered and drift scenarios. Choose the design energy at F = 10⁻⁴/yr. This is consistent with N400's "accidents < 10⁻⁴ may be disregarded".
  - (ii) **Per-class threshold method (AASHTO-style):** choose the smallest vessel class whose impacts (and all larger ones) together stay below the criterion, then design for that class. AASHTO compares the *collapse* frequency (incl. PC) with the threshold; N400 projects typically compare impact frequency or collapse probability (STAPS-cons) with 10⁻⁴.
  - Method (i) is better suited to floating bridges, where capacity depends on impact location (pontoon versus girder) and global dynamics.
- **NTSB effect.** NTSB's endorsement makes AASHTO Method II the de-facto reference for existing-bridge screening in the US. An open engine should therefore output both the IWRAP-style allision frequency and an AASHTO-style AF for comparability.

### Gaps
- The exact N400 (2022/2023 edition) clause numbers and text for ship impact (skipsstøt), including the required ship classes, speeds and any NPRA "rapport" giving a standard method, were **not retrieved**.
- **AASHTO values recalled but unverified in this session:**
  - BR is commonly quoted as 0.6×10⁻⁴ for ships and 1.2×10⁻⁴ for barges (only the 1.2×10⁻⁴ figure was seen in a source).
  - PC = 0.1 + 9(0.1 − H/P) for H/P < 0.1; (1 − H/P)/9 for 0.1 ≤ H/P < 1; 0 for H/P ≥ 1.
  - PG is a normal distribution with σ = LOA about the channel centreline.
  - Impact force P_s = 0.98·√DWT·(V/16) (kips, ft/s).
  - KE = 29.1·C_H·W·V² (kip-ft), with C_H ≈ 1.05–1.25 hydrodynamic mass.
- **Eurocode values not verified:** Table C.4 / Table 4.7 indicative ship masses, speeds and forces, and the Pedersen et al. (1993) bow-force formula in EN 1991-1-7 §C.4. Retrieve them from the EN text before use.
- The NTSB's actual numerical AF for the Key Bridge was not retrieved; only the "≈30× threshold" statement was.

## Q5. Published ship-collision risk analyses and design values for Norwegian floating-bridge projects (Bjørnafjorden etc.)

### Takeaway
Public secondary sources confirm a well-developed but not fully public NPRA process:
- NPRA design-basis documents and dedicated ship-collision risk analyses (Rambøll 2020; SSPA Sweden for E39) exist, alongside Aker Solutions' repair-after-impact study (2021).
- Academic work on Bjørnafjorden ships includes ship-girder/pontoon impact studies (NTNU: Sha, Amdahl, Dørum), energies of 250 MJ (girder, dynamic study) and 1250 MJ (thesis, pontoon near the channel), and the STAPS-cons probability-plus-FEA case study.

Specific design-ship tables and annual frequencies from the K12/K7 reports were not retrieved.

### Cited Findings
- **STAPS-cons (Engineering Structures 2025).** It couples mid-fidelity Monte Carlo allision simulation with FEA consequence results, using AIS data, for a potential floating bridge over **Bjørnafjorden**, to estimate the risk of structural collapse.
  - Of the case-study variants, **three passed and two failed** the Norwegian criterion (annual probability of structural collapse < 10⁻⁴).
  - A 20 % longer missed-turn duration increased both allision frequency and required energy.
  - A less stiff structure reduced demand on the global structure.
  - Source: [Hörteborn et al., Eng. Struct. 2025](https://www.sciencedirect.com/science/article/pii/S0141029625021224); [Chalmers thesis](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf)
- **Site.** Bjørnafjorden is ≈500 m deep and ≈5 km wide at the crossing, and regular **cruise** routes pass the bridge, which raises concern about bridge-girder (deckhouse/superstructure) collisions. — [Chalmers thesis](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf); search summary of the girder-collision literature
- **Girder impact energy.** A dynamic-response study concluded the floating bridge can endure the proposed ship-collision energy of **250 MJ**. For illustration, a 20,000 t ship at 5 m/s gives ≈250 MJ (½mv² = 250 MJ, consistent). — [Sha & Amdahl et al., "Dynamic responses of a floating bridge subjected to ship collision load on bridge girders", Procedia Eng. 2017](https://www.sciencedirect.com/science/article/pii/S1877705817339176/pdf?md5=90677b73675674ae9f057306b7c7173c&pid=1-s2.0-S1877705817339176-main.pdf); [ResearchGate](https://www.researchgate.net/publication/319654389_Dynamic_responses_of_a_floating_bridge_subjected_to_ship_collision_load_on_bridge_girders)
- **Pontoon impact energy.** A master's thesis analysed a **1250 MJ** ship impact on the pontoon nearest the navigation channel, at several damping levels. This is a thesis case, not confirmed as the design basis. — [Eldegard, "Analysis and Design of Floating Bridge over Bjørnefjorden – Floating Bridge Subjected to Large Ship Collisions"](https://www.semanticscholar.org/paper/Analysis-and-Design-of-Floating-Bridge-over-Bridge-Eldegard/a9f8f1f4f101c6c1c65b0255618f5413112a07f6)
- **Analysis method (OMAE 2017, NTNU).** Ship–pontoon and ship–girder collisions were analysed with local LS-DYNA models and global USFOS response. Sha, Amdahl and Dørum used a two-step approach: ship against a rigid pier, then the force applied to the global bridge. Results from different codes show significant discrepancies. — [OMAE2017-61620](https://asmedigitalcollection.asme.org/OMAE/proceedings-abstract/OMAE2017/57779/V009T12A021/279310); [Design of floating bridge girders against accidental ship collision loads](https://www.researchgate.net/publication/308614736_Design_of_floating_bridge_girders_against_accidental_ship_collision_loads)
- **Reference trail.** A 2024/25 Springer chapter cites:
  - NPRA (2023) *Design basis Bjørnafjorden: end-anchored floating bridge*;
  - Rambøll (2020) ship-collision analysis;
  - Aker Solutions (2021) *Repair after ship impact study, Bjørnafjorden floating bridge*.
  - Pontoons must be replaceable, and the structure must survive a collision with a big ship with full traffic on the bridge.
  - Source: [Analysis and Design of Floating Bridges: Recent Advances and Challenges](https://link.springer.com/chapter/10.1007/978-981-96-5435-2_5)
- **SSPA Sweden.** E39 programme material lists "risk analyses for ship collisions: SSPA Sweden AB". — [E39 presentation, Embassy of Portugal](https://www.cpci.pt/wp-content/uploads/2015/03/Autoestrada_E-39.pdf)
- **Mitigation focus.** Askeland et al. (2021, NPRA/COWI/DTU) assessed risk-reducing measures for extreme fjord crossings; see Q2. — [DTU Orbit](https://orbit.dtu.dk/en/publications/risk-assessment-of-ship-allision-in-extreme-fjord-crossings/)

### Inferences
- **Pontoon versus girder.** Bjørnafjorden practice treats pontoon (bow) impact and girder (deckhouse or superstructure) impact as separate scenarios with separate energies. The girder scenario is governed by tall ships (cruise) and air-draft clearance. An allision engine for floating bridges must therefore carry ship height and air-draft per class and filter by structure elevation, as OMRAT does (structure height < ship height).
- **Sensitivity of the energy demand.** The gap between ~250 MJ (girder study) and 1250 MJ (pontoon thesis) shows that design energy is extremely sensitive to the vessel population and speed assumptions. Deriving it from an exceedance curve at 10⁻⁴/yr (see Q4) is necessary.

### Gaps
- The Bjørnafjorden K12/K7 ship-collision risk reports were not retrieved: design vessels per scenario, annual frequencies per vessel class, design impact energies adopted in the NPRA design basis, consultant names (Norconsult/AFRY/Aas-Jakobsen/Safetec/DNV) and the evaluated mitigation measures (VTS, collision warning, channel layout).
- No public data were found for **Halsafjorden or Sulafjorden** ship-collision analyses; the searches were not run because of the budget limit.

## Q6. Theses and papers on AIS-based allision risk for Norwegian fjord crossings (incl. Nordfjorden)

### Takeaway
The identifiable AIS-based allision work for Scandinavian/Norwegian crossings is the Hörteborn (Chalmers/RISE) series, co-authored with NPRA's Mathias Eidem:
- STAPS in Ocean Engineering 2021;
- the 2021 licentiate thesis;
- the IWRAP verification at ICCGS 2023;
- the IWRAP–STAPS comparison in RESS 2025;
- Hörteborn & Eidem in J. Navigation 2025 (15 Scandinavian bridges);
- STAPS-cons on Bjørnafjorden in Engineering Structures 2025;
- the PIANC 2025 US study.

These sit alongside NTNU structural-impact theses on Bjørnafjorden and the Hassel/Utne/Vinnem passing-vessel allision model for the Norwegian shelf. No Nordfjorden-specific allision study was found.

### Cited Findings
- STAPS: a method for risk analysis of ship collisions with stationary infrastructure using AIS data and a ship-manoeuvring simulator. — [Ocean Engineering 2021](https://www.sciencedirect.com/science/article/pii/S0029801821008088); [Licentiate thesis "Ship Behaviour and Ship Bridge Allision Analysis" (2021)](https://infrasweden.nu/wp-content/uploads/2024/07/Ship-Behaviour-and-Ship-Bridge-Allision-Analysis.-Licentiate-Thesis-2021.pdf)
- Hörteborn & Eidem (J. Navigation 2025): 15 Scandinavian bridges, 6 years of AIS and incident data. — [Cambridge Core](https://www.cambridge.org/core/journals/journal-of-navigation/article/probability-of-active-navigational-failures-incident-analyses-for-use-in-shipbridge-allision-risk-assessments/4457B860D16A6EF1C8E0B570E047A9EB)
- Hörteborn PhD-level synthesis, "Risk analysis methodology for ship-bridge allisions". — [Chalmers full text](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf)
- OMRAT: an open QGIS tool with AIS ingestion (database setup, AIS connection widgets), lateral-distribution fitting and IWRAP XML round-trip. — [OMRAT GitHub](https://github.com/axelande/OMRAT); [OMRAT user guide](https://axelande.github.io/OMRAT/user_guide.html)
- Hassel, Utne & Vinnem (2021): an allision risk model for passing vessels and offshore oil and gas installations on the Norwegian Continental Shelf. — [Proc IMechE Part O](https://journals.sagepub.com/doi/abs/10.1177/1748006X20957481)
- NTNU structural theses and papers on Bjørnafjorden large-ship collisions: Eldegard; Sha/Amdahl/Dørum. — [Semantic Scholar](https://www.semanticscholar.org/paper/Analysis-and-Design-of-Floating-Bridge-over-Bridge-Eldegard/a9f8f1f4f101c6c1c65b0255618f5413112a07f6); [OMAE 2017](https://asmedigitalcollection.asme.org/OMAE/proceedings-abstract/OMAE2017/57779/V009T12A021/279310)

### Inferences
- For a Nordfjorden product, the closest methodological precedent is STAPS/OMRAT applied to AIS data, calibrated with NPRA incident data in the way Hörteborn & Eidem did. The NPRA co-authorship suggests NPRA is receptive to AIS-calibrated, site-specific causation probabilities rather than IWRAP defaults.

### Gaps
- No NTNU Open or DTU master's thesis on AIS-based allision risk specifically for a Norwegian fjord crossing, or on **Nordfjorden**, was found. NTNU Open was unreachable (DNS failure) and the search budget was exhausted, so this is "not found", not "does not exist".
- Recommended follow-up queries: "skipsstøt Nordfjorden", "Nordfjordbrua" or "E39 Nordfjord kryssing", NTNU Open "allision AIS fjord", and Kystverket "risikoanalyse Nordfjorden".
