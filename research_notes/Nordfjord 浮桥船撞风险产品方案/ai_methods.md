# AI/ML and Data-Driven Methods for Ship–Bridge Allision Risk Assessment, Mitigation Optimization and Real-Time Collision Warning (state of the art 2018–2026)

> Method note: WebFetch and curl were blocked (DNS failure or proxy 403) for arxiv.org, sciencedirect, mdpi, ncbi, ascelibrary, repec and most vendor sites. Every finding below therefore comes from search-engine abstracts and snippets, not full texts. Figures are as reported by the authors in abstracts and highlights. Treat all numbers as "author-reported, unreplicated" unless stated otherwise.
>
> Maturity labels used throughout:
> - **M (Mature/operational):** used in practice or standard in consultancy and authority work.
> - **P (Proven in research):** many peer-reviewed papers with consistent results; ready to engineer into a product.
> - **R (Research-grade/experimental):** few papers, preprints, fragile benchmarks or no field validation.

---

## 1. AIS trajectory analytics: cleaning/segmentation, route extraction, lateral distributions, ship-type classification, trajectory prediction, anomaly / loss-of-propulsion detection

### Takeaway
The front end of an allision engine is mature and largely open-source: AIS cleaning and segmentation, density-based route and waypoint extraction, and empirical lateral distributions (KDE/GMM). Deep trajectory prediction (LSTM, Transformer/TrAISformer, GNN, LLM-based) is research-grade for safety use. A 2026 leakage-aware audit shows that evaluation protocol shifts reported errors by as much as architecture does, and errors explode out of region. No dedicated, validated ML detector for loss of propulsion from AIS was found. Drift modelling is still physics-led (for example OpenDrift), with hybrid physics–NN drift models only just emerging.

### Cited Findings

**Cleaning / segmentation (M)**
- AISdb (2024) is an open-source Python library. It ingests a real-time stream, NMEA (NM4) or CSV, removes noise, interpolates, and segments trajectories by speed and temporal-gap thresholds. — [AISdb, arXiv 2407.08082](https://arxiv.org/pdf/2407.08082)
- The "α-method" (Paulig & Okhrin, Ocean Engineering 2024) is an open-source framework that splits raw AIS into trajectories. It uses α-quantiles of empirical feature distributions, turning rates, distances, and ship length as a manoeuvrability proxy. — [TU Dresden record](https://fis.tu-dresden.de/portal/en/publications/an-opensource-framework-for-datadriven-trajectory-extraction-from-ais-datathe-method(9d36f801-cc91-4168-a7b1-0ed83f31d842).html); [arXiv 2407.04402](https://arxiv.org/abs/2407.04402v2)
- PyMove, scikit-mobility and PTRAIL handle interpolation, outlier removal and feature extraction but were not built for AIS. — [AISdb paper](https://arxiv.org/html/2407.08082v1)

**Route extraction (M/P)**
- TREAD is an unsupervised maritime route-extraction framework aimed at anomaly detection and route prediction. It addresses AIS intermittency and persistence problems. — [arXiv 2403.05778](https://arxiv.org/pdf/2403.05778)
- Graph-based route extraction turns trajectories into semantic trip objects and builds a directed traffic graph. It works well for merchant ships but struggles with vessels whose behaviour changes often, such as fishing vessels. — [arXiv 2404.00902](https://arxiv.org/pdf/2404.00902)
- Applications of DBSCAN to AIS:
  - Waypoint discovery, compared against genetic-algorithm and ant-colony methods, with a streaming-data constraint (Univ. Twente). — [U Twente](https://research.utwente.nl/en/publications/using-machine-learning-for-unsupervised-maritime-waypoint-discove/)
  - Clustering of turning points into turning sections (IST Lisbon). — [monitraffic](https://monitraffic.tecnico.ulisboa.pt/?p=793)
  - Clustering of velocity-change points into the vertices of an unsupervised route graph, with sea-lanes as edges (Univ. Campania). — [iris.unicampania](https://iris.unicampania.it/handle/11591/389932)
- Other clustering variants:
  - OPTICS, which needs no preset cluster count. — [monitraffic](https://monitraffic.tecnico.ulisboa.pt/?p=793)
  - HDBSCAN with Hausdorff distance. — [MDPI 2130602](https://mdpi.com/2130602)
  - Iterative density-based clustering of standard port-to-port routes (2025), which needs no ground truth and reports under 5% outliers. — [arXiv 2503.22734](https://arxiv.org/html/2503.22734v1)

**Lateral distribution estimation (M/P)**
- IWRAP's route-lateral-distribution output, derived from AIS, has been used to compare traffic before and after offshore wind farm (OWF) construction. — [ResearchGate 333603301](https://www.researchgate.net/publication/333603301_Assessment_of_the_Influence_of_Offshore_Wind_Farms_on_Ship_Traffic_Flow_Based_on_AIS_Data)
- Kernel Density Estimation on historical AIS has been used to identify channel boundaries, bridge-pier domains and ship domains in inland bridge waterways. A Velocity Obstacle step then flags ship–bridge collision candidates. — [Ocean Eng. 2022, S0029801822024209](https://www.sciencedirect.com/science/article/abs/pii/S0029801822024209)
- A 2024 inland-waterway study fits Gaussian mixtures to lateral offsets from the fairway centre, conditioned on river discharge, and feeds them as features to a Transformer predictor. — [arXiv 2406.02344](https://arxiv.org/html/2406.02344v2)
- Dalsnes et al. (2018) use GMM-based "neighbour course distributions" from AIS to predict position 5–15 min ahead, with uncertainty and multimodality. — [IEEE 8455607](https://ieeexplore.ieee.org/iel7/8442112/8454975/08455607.pdf)
- For mixture-distributed positions, collision probability is a weighted sum over component pairs. This is shown in the space-debris literature and carries over directly to vessel–pier geometry. — [arXiv 2405.12230](https://arxiv.org/pdf/2405.12230)

**Ship-type classification (P)**
- Reported accuracy depends heavily on the number of classes:
  - 14 fine-grained classes: about 78% accuracy for a Transformer+LightGBM hybrid on 2,196 real trajectories, versus about 65% for Transformer alone and about 67% for LightGBM alone. — [BIT 2026](https://pure.bit.edu.cn/en/publications/feature-transformer-and-lightgbm-ensemble-for-ship-trajectory-rec/)
  - 4 classes: average F1 of about 97.5% with HMM/DNN/CNN. — [Remote Sensing 16:4245 (2024)](https://mdpi-res.com/d_attachment/remotesensing/remotesensing-16-04245/article_deploy/remotesensing-16-04245.pdf)
  - AISFusionNet, which fuses time series with trajectory images: 95.6% accuracy. — [TRID 2729074](https://trid.trb.org/View/2729074)
- XGBoost/RF reach about 97% on cargo subtypes in the Changhua wind farm channel. The authors note that the AIS ship-type code is too coarse to separate bulk carriers from container ships. — [NTOU](https://scholars.ntou.edu.tw/handle/123456789/24634?locale=en)

**Trajectory prediction (P for short horizons, R for safety-critical claims)**
- TrAISformer (Nguyen & Fablet; arXiv 2021, IEEE Access 2024):
  - Method: discretised high-dimensional AIS encoding with a cross-entropy loss, which gives multimodal (generative) predictions.
  - Result: mean error below 10 nmi up to about 10 h, on Danish Maritime Authority AIS tested on 21–31 March 2019 with horizons up to 15 h.
  - — [arXiv 2109.03958](https://arxiv.org/pdf/2109.03958); [DOAJ/IEEE Access](https://doaj.org/article/81f207ff824b40f6a0bcf97e0de0d77e)
- Leakage-aware audit (Raisi & Nazarzehi Had, arXiv Sept 2026) of TrAISformer, GATransformer and AISFormer-type models on 31 days of Danish and 30 days of US Gulf AIS:
  - A best-of-16 oracle decoder lowers error by 2.1–3.2× versus greedy decoding.
  - Sharing vessels across train and test lowers greedy error by 23–25% at 1 h.
  - A region-disjoint split raises TrAISformer's 1-h error from 2.2 km to 24.6 km.
  - GATransformer's graph attention gives no measurable benefit; its waterway feature is worth 12–22%.
  - — [arXiv 2609.25827](https://arxiv.org/pdf/2609.25827); [prismix summary](https://prismix.dev/news/c05a989350e2)
- 2025 review of Transformer-based AIS monitoring (prediction, anomaly detection, collision risk). — [arXiv 2505.07374](https://arxiv.org/pdf/2505.07374)
- GNN and multi-vessel models:
  - Attention-based spatiotemporal GCN (2024) and G-STGAN (2023), cited in Frontiers 2025. — [Frontiers fmars.2025.1605216](https://www.frontiersin.org/journals/marine-science/articles/10.3389/fmars.2025.1605216/full)
  - STMGF-Net, a spatiotemporal multi-graph fusion model (IEEE T-ITS 2024). — [same](https://www.frontiersin.org/journals/marine-science/articles/10.3389/fmars.2025.1605216/full)
  - Interaction-aware multi-vessel prediction with collision risk. — [arXiv 2509.01836](https://arxiv.org/pdf/2509.01836)
- Other directions:
  - Physics-informed NN vessel prediction using finite-difference kinematics. — [arXiv 2506.12029](https://arxiv.org/pdf/2506.12029)
  - Reliability comparison of predictors via Probability of Detection. — [arXiv 2508.14198](https://arxiv.org/pdf/2508.14198)
  - Systematic review of deep-learning ship trajectory prediction (JMSE 2026). — [doi 10.3390/jmse14090810](https://doi.org/10.3390/jmse14090810)
- LLM-based trajectory work (R):
  - AIS-LLM: a unified framework for prediction, anomaly detection and collision risk. — [arXiv 2508.07668](https://arxiv.org/pdf/2508.07668)
  - ShipTraj-R1: GRPO-tuned LLM. — [arXiv 2603.02939](https://arxiv.org/pdf/2603.02939)
  - Reasoning-LLM long-horizon and destination forecasting. — [arXiv 2606.08633](https://arxiv.org/pdf/2606.08633)
- Du et al. (RESS 2026) use an LSTM + Unscented Kalman Filter (LSTM-UKF) as a micro-scale predictor feeding a domain-violation risk model. — [ScienceDirect S0951832025007112](https://www.sciencedirect.com/science/article/abs/pii/S0951832025007112)

**Anomaly detection (P for generic, R for loss of propulsion)**
- A multi-class ANN separates intentional from unintentional AIS on-off switching, flagging consecutive dropouts above a threshold. — [arXiv 2002.05013](https://arxiv.org/pdf/2002.05013)
- Evidential deep learning on RNN predictions flags unusual turns and loss of signal using class probabilities with uncertainty. — [arXiv 2107.01557](https://arxiv.org/html/2107.01557)
- Isolation Forest is widely used on speed, position-jump, time-gap and turn-angle features. — [arXiv 2505.07374](https://arxiv.org/pdf/2505.07374)
- Surveys and related work:
  - A comprehensive survey of ML anomaly detection in ship behaviour (Eng. Appl. AI 2025). — [S0952197625026272](https://www.sciencedirect.com/science/article/abs/pii/S0952197625026272)
  - A new unsupervised-evaluation metric, MADQI (2026). — [arXiv 2605.30388](https://arxiv.org/pdf/2605.30388)
  - Equation-grounded synthetic anomalies for training (2026). — [arXiv 2606.29721](https://arxiv.org/pdf/2606.29721)
  - MLOps for self-adaptive anomaly detection under data drift. — [JMSE 14:1152](https://doi.org/10.3390/jmse14131152)
- The VTS anomaly-detection survey (around 2023) stated that ML-based systems "have not found their way into VTS systems yet". — [ResearchGate 371334928](https://www.researchgate.net/publication/371334928_Maritime_Anomaly_Detection_for_Vessel_Traffic_Services_A_Survey)
- Drifting / loss of propulsion:
  - TU Delft models drifting-ship allision probability with OpenDrift (wind, wave and current forcing) combined with ship density. — [TU Delft repository](https://repository.tudelft.nl/record/uuid:cff6c8fc-9608-4e13-92c0-80a0201545bb)
  - A 2025 hybrid physical–neural drift model uses ConvLSTM-forecast wind and current fields. Its authors argue that pure ML lacks interpretability and fails in data-sparse areas. — [Ocean Eng. S0029801825032421](https://www.sciencedirect.com/science/article/abs/pii/S0029801825032421)
- Chalmers (2025 dissertation) proposes STAPS-cons (Ship Traffic Allision Probability using Simulations – consequences). It combines AIS-derived ship-failure statistics, Monte Carlo simulation and consequence analysis. — [Chalmers 549035](https://research.chalmers.se/publication/549035/file/549035_Fulltext.pdf); [TRID 2751949](https://trid.trb.org/View/2751949); [Chalmers 521842](https://research.chalmers.se/en/publication/521842)

### Inferences
- **Fit with the physics engine:**
  - AIS cleaning, segmentation and route extraction give leg and waypoint geometry plus traffic counts per ship class.
  - KDE or GMM lateral distributions replace IWRAP-style "normal + uniform" assumptions and feed the geometric collision probability for each pier or pontoon directly.
  - Ship-type and size classification fills coarse or missing AIS static fields, which matters for impact-energy distributions.
  - These are all **M/P** and low-risk to productise.
- **Where trajectory prediction belongs:** it is a real-time warning component, not a frequency-estimation one. Given the 2026 leakage audit, any product claim should be validated on vessel-, time- and region-disjoint splits at the target site. Use calibrated probabilistic outputs (for example a GMM or best-of-N with probability estimates), not point forecasts.
- **Loss of propulsion:** no method was found that detects propulsion loss specifically. A practical detector would combine:
  - AIS gap detection;
  - a speed-through-water proxy against SOG/COG;
  - ROT and heading variability;
  - drift-consistency checks against wind and current, via a physics drift model such as OpenDrift.

  The drift-model approach matters most for a floating bridge in a fjord, where drifting allision may dominate.

### Gaps
- TRACLUS applied to AIS: no source retrieved. It is commonly cited in the literature, but could not be verified here.
- No full-text per-horizon error tables were retrieved for TrAISformer versus LSTM baselines; only the "<10 nmi up to about 10 h" headline and the 2026 audit figures are available.
- No validated AIS-only loss-of-propulsion or blackout detection paper, with labelled events and reported precision and recall, was found.
- No source quantified how much KDE or GMM lateral distributions change IWRAP allision frequencies compared with the default parametric assumptions.

---

## 2. Data-driven conflict / near-miss detection, EVT and Bayesian extrapolation to accident frequency, and validation of IWRAP-type models with AIS

### Takeaway
AIS near-miss detection is proven methodology (CPA/TCPA, ship-domain overlap, DDV/TDV, rate-of-turn evasive-manoeuvre filters). Bridge-specific variants exist for Chinese inland bridge waters: AIS-derived ship and pier domains, KDE plus Velocity Obstacle, and a safety potential field. EVT extrapolation of near-misses to collision frequency is research-grade, with essentially one recent RESS paper (Du et al. 2026). Validation of IWRAP-type models against AIS near-miss counts has only been done in a few studies, mostly Gulf of Finland (Goerlandt et al.) and Norway (Hassel et al.). These studies report large manual-review disagreement rates, which undermines automated near-miss counts as ground truth.

### Cited Findings

**Near-miss detection**
- Zhang et al. (2015) is the foundational AIS near-miss method, ranking conflicts using ship domain, distance, relative speed and bearing (VCRO). A later extension adds a Ship Domain Overlap Index, tested in the Malacca Strait. — [RG 280920565](https://www.researchgate.net/publication/280920565_A_method_for_detecting_possible_near_miss_ship_collisions_from_AIS_data); [RG 305824363](https://www.researchgate.net/publication/305824363_An_advanced_method_for_detecting_possible_near_miss_ship_collisions_from_AIS_data)
- Hassel, Grossmann, Aalberg & Arntsen (ESREL 2020; SINTEF) define a near-collision as a domain violation plus a last-moment evasive manoeuvre, detected as abnormal ROT.
  - Over 3 years in Vestfjorden (Norway) they found 476 domain violations and 46 near-collisions.
  - Manual review confirmed 19 of 47 flagged situations.
  - A follow-up applies a COLREGs-based ROT test in open waters.
  - — [ESREL 2020 PDF](https://www.rpsonline.com.sg/proceedings/esrel2020/pdf/3520.pdf); [SINTEF](https://www.sintef.no/en/publications/publication/1892623/)
- DDV (Degree of Domain Violation) and TDV (Time to Domain Violation) underpin a real-time collision-risk perception model. — [Ocean Eng. 2022 S0029801822018376](https://www.sciencedirect.com/science/article/abs/pii/S0029801822018376)
- A ship-domain-based collision-risk model has been proposed for near-miss detection and Collision Alert Systems. — [RESS 2021 S0951832021002945](https://www.sciencedirect.com/science/article/abs/pii/S0951832021002945)
- A 2026 inland study calibrates the avoidance-triggering domain boundary from AIS-extracted manoeuvre onsets. It stresses that domain overlap is a severity surrogate, not a collision probability. — [JMSE 14:1868](https://doi.org/10.3390/jmse14201868)

**Bridge / fixed-structure near-allision**
- An AIS data-driven ship-domain model for bridge waters (multi-bridge Yangtze reach at Wuhan) uses ellipse domains and fuzzy quaternion domains for inland ships and piers. Domain size tracks speed and length. — [J. Navigation](https://www.cambridge.org/core/journals/journal-of-navigation/article/abs/automatic-identification-system-datadriven-model-for-analysis-of-ship-domain-near-bridgewaters/C306A95CE4593D1521F59524A8830290)
- Ship–bridge collision candidate detection uses KDE of AIS for the channel and pier domains, plus a Velocity Obstacle step. — [Ocean Eng. 2022](https://www.sciencedirect.com/science/article/abs/pii/S0029801822024209)
- A safety-potential-field warning model for inland bridge areas was tested in simulation at the Baijusi Bridge. — [Appl. Sci. 14:9089 (2024)](https://www.mdpi.com/2076-3417/14/19/9089)
- Other bridge-area work:
  - Ship–bridge collision risk algorithm that accounts for ship dimensions. — [JMSE 10:1635 (2022)](https://doi.org/10.3390/jmse10111635)
  - AIS ship distributions combined with nonlinear FE for bridge-area risk. — [De Gruyter 2022](https://www.degruyterbrill.com/document/doi/10.1515/nleng-2022-0324/html?lang=en)
- A Johns Hopkins preprint (April 2026) screens US bridges with AIS and finds ship-strike risk high for 54 bridges. It assesses allision risk, not collapse probability. — [engrXiv 6846](https://engrxiv.org/preprint/view/6846)
- NTSB (March 2025) recommended vulnerability assessments for 68 bridges (30 owners, 19 states). Its report MIR-25-10 says assessments should use current vessel traffic data. — [NTSB press release](https://www.ntsb.gov/news/press-releases/Pages/nr20250320.aspx); [NTSB MIR2510](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2510.pdf)

**EVT / Bayesian extrapolation**
- Du et al. (RESS vol. 265, 2026) take a multi-scale approach in restricted waters:
  - Micro scale: LSTM-UKF trajectory prediction inside a DDV model to find high-risk multi-ship clusters.
  - Macro scale: EVT on the tail of DDV to estimate collision probability and frequency.
  - Huangpu River AIS case: collision probability 0.21% and annual frequency 11.54 incidents (as reported).
  - — [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0951832025007112); [IDEAS](https://ideas.repec.org/a/eee/reensy/v265y2026ipas0951832025007112.html)
- Goerlandt et al. (Gulf of Finland) compared a geometric collision-candidate model against AIS-derived near-collisions and historical accidents to estimate a causation factor. — [TransNav](https://transnav.eu/files/html_file.php?id_artykuly=306); [RG 256968743](https://www.researchgate.net/publication/256968743_Analysis_of_near_collisions_in_the_Gulf_of_Finland)
- Using a Bayesian network (BN) for causation, the average causation probability was about 2.7×10⁻⁴ for crossings and 1.0×10⁻⁵ for head-on encounters (July 2006 AIS). — [academia.edu](https://www.academia.edu/54124761/Accident_probabilities_in_selected_areas_of_the_Gulf_of_Finland)
- Hänninen & Kujala found collision probability "very sensitive" to the causation probability. — [TransNav](https://www.transnav.eu/Article_The_Effects_of_Causation_Probability_H%C3%A4nninen,13,207.html)
- A US Atlantic Coast study (pre-OWF) used 1 year of AIS to count encounters and elliptical-domain near-misses, and estimated collision rates and causation probabilities. — [Ocean Eng. 2022](https://www.sciencedirect.com/science/article/abs/pii/S0029801822015451); [OSTI 1970466](https://www.osti.gov/servlets/purl/1970466)

**IWRAP + AIS**
- IWRAP is the IALA-recognised collision, allision and grounding frequency model used by USCG for Navigation Safety Risk Assessments. — [USCG NAVCEN WI 2022-01](https://www.navcen.uscg.gov/sites/default/files/pdf/waterways/nsra/NAVCEN%20Work%20Instruction%202022-01%20v2.pdf)
- IWRAP Mk II with AIS from a site receiver was used at Donghae-1 (Korea) to derive the optimal traffic width when passing an OWF. — [Ocean Eng. 2024 S0029801824028361](https://www.sciencedirect.com/science/article/abs/pii/S0029801824028361)
- A 2021 review of collision-frequency models found that causation probabilities are often borrowed from other areas, and that validation is hard because outputs cannot easily be checked against historical counts. — [JMSE 9(5):533](https://www2.mdpi.com/2077-1312/9/5/533)
- A 2025 OWF review notes that before/after AIS comparisons show rerouting and more frequent near-misses. — [JMSE 13:1330](https://doi.org/10.3390/jmse13071330)

### Inferences
- **Calibrating causation for allision:** a near-allision detector can be used for Bayesian updating of the causation probability (Pc) and of failure-event rates. Candidate signals are:
  - pier or pontoon domain violation;
  - TDV below a threshold;
  - an abnormal-ROT evasive manoeuvre.

  The prior comes from IWRAP or AASHTO-type defaults; the likelihood comes from local near-allision counts. This is a natural use of site AIS for a floating bridge.
- **EVT:** EVT on DDV or minimum-distance tails is the most direct "near-miss → frequency" route. With a single main source, it should be offered as an R-grade cross-check, not as the primary frequency estimate.
- **Detector precision:** the Hassel et al. manual-review result (19 of 47 confirmed) implies roughly 40% precision for automated near-collision flags. Any product should include a human or LLM-assisted review loop and report detector precision.

### Gaps
- No paper was found that formally validates IWRAP allision frequencies against observed near-allision counts at a bridge.
- EVT threshold-selection and goodness-of-fit details, and the use of block-maxima versus POT, were not available from the Du et al. abstract.
- No Bayesian hierarchical model was found that pools near-allision data across multiple bridges.

---

## 3. Causation probability via Bayesian networks and ML on accident databases; human/organisational factors; transfer to VTS effectiveness

### Takeaway
Data-driven BNs (TAN, structure learning by PC, GTT or BS with EM) trained on accident-investigation databases are mature research tools for factor importance and consequence prediction. They do not directly yield site-specific causation probability. Supervised ML (XGBoost, RF, AutoML) predicts spatial accident occurrence with high AUC. LLM- and NLP-based knowledge-graph extraction from accident reports now reaches F1 of about 0.87–0.95. Quantified VTS effectiveness remains largely expert-elicited and dated (around 40–60% reduction), with no rigorous causal before/after evaluation found.

### Cited Findings
- A global data-driven Tree-Augmented Naive Bayes model (RESS 2026), built from collision investigation reports covering 1978–2024 from multiple bodies, models both vessels jointly. It reports over 93% accuracy predicting collision consequences. — [RESS S0951832026007702](https://www.sciencedirect.com/science/article/pii/S0951832026007702)
- A physics-knowledge-integrated BN compared Bayesian search, Greedy Thick Thinning and PC structure learning, each with EM parameter learning. — [RG 383063532](https://www.researchgate.net/publication/383063532_Analyzing_risk_influencing_factors_of_ship_collision_accidents_A_data-driven_Bayesian_network_model_integrating_physical_knowledge)
- Related BN applications:
  - Multi-source BN for high-sea accidents by accident type. — [RG 393513760](https://www.researchgate.net/publication/393513760_Multi-source_data-driven_Bayesian_network_for_risk_analysis_of_maritime_accidents_in_the_high_sea)
  - Data-driven BN for accident severity. — [Ocean Eng. 2022](https://www.sciencedirect.com/science/article/abs/pii/S0029801822028463)
- An offshore-platform allision BN built on 557 incidents found human error to be the leading contributor (30%). — [J. Marine Sci. Appl. 2026](https://link.springer.com/article/10.1007/s11804-026-00812-1)
- The Jintang Bridge (China) study (Ma et al., Sensors 2022) combines AIS, Monte Carlo and BN. Reported collision-avoidance failure probabilities:

  | Measures | Failure probability |
  |---|---|
  | None | about 2.2×10⁻³ |
  | Active | about 1.6×10⁻³ |
  | Passive | about 1.1×10⁻⁴ |
  | Both | about 8×10⁻⁵ |

  Passive measures such as arresting cables are needed where active-measure response time is long. — [Sensors 22:2857](https://doi.org/10.3390/s22082857); [PMC9025040](https://pmc.ncbi.nlm.nih.gov/articles/PMC9025040)
- Rawson & Brito (Transport Reviews 2022) surveyed 76 supervised-ML maritime-risk papers and flagged data challenges. — [RG 358436937](https://www.researchgate.net/publication/358436937_A_survey_of_the_opportunities_and_challenges_of_supervised_machine_learning_in_maritime_risk_analysis)
- Rawson, Brito & Sabeur (Risk Analysis 2022) built spatial ML models of UK collision and grounding risk with accuracy and AUC above 90% in most cases; XGBoost and RF performed best. — [Bournemouth eprint](https://eprints.bournemouth.ac.uk/37810/1/Risk%20Analysis%20-%202021%20-%20Rawson%20-%20Spatial%20Modeling%20of%20Maritime%20Risk%20Using%20Machine%20Learning.pdf)
- Other accident-prediction ML:
  - AutoML for accident risk. — [RG 379995987](https://www.researchgate.net/publication/379995987_Predicting_maritime_accident_risk_using_Automated_Machine_Learning)
  - Selective ensemble with SHAP for human-related accident types. — [Heliyon 2024](https://www.cell.com/heliyon/fulltext/S2405-8440(24)06077-8)
  - Spatial ML risk for China offshore waters. — [Ocean Coast. Manag. 2023](https://www.sciencedirect.com/science/article/abs/pii/S0964569123004593)
  - Long-term vessel incident risk factors. — [arXiv 2405.19804](https://arxiv.org/pdf/2405.19804)
  - Busan port: XGBoost, RF, MLP and SVM on accident reports plus VTS data (Nov 2018–Jul 2024). — [Ocean Eng. 2024 S0029801824033067](https://www.sciencedirect.com/science/article/abs/pii/S0029801824033067)
- Accident-report mining (P/R):
  - MAKG: built from 581 China MSA reports; extraction F1 0.910 ±0.006, reported as 91.2% and an 8.44% gain over baseline. — [Ocean Eng. 2024 S0029801824026180](https://www.sciencedirect.com/science/article/pii/S0029801824026180)
  - Ontology-constrained LLM prompting (RESS 2026) on 318 Chinese collision reports: NER F1 94.7%, most relation types above 96% F1. — [RESS S0951832026009579](https://www.sciencedirect.com/science/article/abs/pii/S0951832026009579)
  - Collision-report semantic modelling (JMSE 2026): mean F1 86.70%, but the rare "Recommendation" class scores only 57.5%. — [JMSE 14:448](https://doi.org/10.3390/jmse14050448)
  - LLM graph extraction from UK (1990–2023) and China (2016–2024) reports. — [Ocean Coast. Manag. 2025](https://www.sciencedirect.com/science/article/abs/pii/S0964569125004788)
  - MaRAG: KG + RAG question answering. — [Ocean Eng. 2025](https://www.sciencedirect.com/science/article/abs/pii/S0029801825033049)
  - RAG on MAIB reports comparing 9 LLMs against 18 prompt-injection techniques. — [JMSE 14:983](https://doi.org/10.3390/jmse14110983)
- VTS effectiveness:
  - An expert-elicited, ship-based study puts maximum VTS benefit at 60% for collisions and 55% for groundings, with average effectiveness of 43.3%. Its casualty-rate reduction factor for radar VTS with automatic track analysis is 0.50–0.70. — [RG 320622741](https://www.researchgate.net/publication/320622741_A_Ship_Based_Approach_to_Determine_the_Effectiveness_of_VTS_Systems_in_Reducing_Vessel_Accidents)
  - A BN estimate puts the risk-reducing effect of shipboard AIS at about 55%. — [SDU](https://portal.findresearcher.sdu.dk/en/publications/risk-reduction-effect-of-ais-implementation-on-collision-risk/)

### Inferences
- **Role of BNs in the engine:** BNs act as the Pc modifier layer. Nodes such as visibility, pilotage, VTS monitoring, bridge-warning-system state, fatigue and ship type multiply or condition the baseline Pc. Data-driven BN structure from accident databases (TAN, PC) can propose the structure; engineers should fix it and calibrate locally with near-miss data from Section 2.
- **LLM report mining:** knowledge-graph extraction is mature enough (F1 about 0.9) to build and maintain a human/organisational-factor evidence base that updates BN priors. Rare-class performance (about 57%) means recommendations and organisational factors still need human checking.
- **VTS and warning credit:** VTS or active-warning risk credit in the engine should be parameterised as an uncertain reduction factor:
  - for VTS, roughly 0.4–0.7 per the dated expert studies;
  - for active bridge warning in the Jintang model, about 27% (2.2→1.6×10⁻³).

  Both should be treated as epistemic ranges for sensitivity analysis, not point values.

### Gaps
- No study was found that learns allision-specific (rather than ship–ship) causation probability from accident databases.
- No causal before/after or difference-in-differences evaluation of VTS or bridge-warning systems using AIS plus casualty data was found.
- Contemporary (2020+) quantified VTS effectiveness figures were not found.

---

## 4. Surrogate models for risk engines and structural FE, active learning, UQ, Bayesian optimization, and NSGA-II for risk-control-option portfolios

### Takeaway
Surrogates of ship/barge–pier impact FE (Kriging, RBF/BP-NN, RF/FFNN, GP) are proven in research for impact-force prediction. Active-learning PC-Kriging and AK-MCS are mature in structural reliability, and one paper applies them to vessel impact on an offshore wind substructure. Multi-objective optimisation in maritime work is mostly applied to collision-avoidance manoeuvres, search-and-rescue allocation and sensor or camera placement. No published work was found that optimises a portfolio of ship–bridge risk-control options with NSGA-II or Bayesian optimisation; that remains a white space.

### Cited Findings
- **Barge–pier impact surrogates:** a framework uses validated nonlinear FE over barge weight × velocity, with Kriging and NN surrogates predicting impact-force time histories. — [Ocean Eng. 2023 S0029801823017316](https://www.sciencedirect.com/science/article/abs/pii/S0029801823017316)
- **Early RBF surrogate:** an RBF network trained on FE of the Nanjing Yangtze River Bridge maps ship velocity and mass to collision force, motivated by FE being "time-consuming and requiring supercomputing". — [RG 225507367](https://www.researchgate.net/publication/225507367_Calculation_method_of_ship_collision_force_on_bridge_using_artificial_neural_network)
- **Peak-force NN:** a peak impact-force neural network (Korea, 2022). — [KoreaScience](https://www.koreascience.kr/article/JAKO202210351649966.page)
- **Hybrid FE–ML (Haq et al., Struct. Control Health Monit. 2026):** FE over tonnage, speed, angle and pier geometry, with RF and FFNN surrogates for real-time consequence prediction. The snippet reports peak force rising from about 0.8 to 3.5 MN as speed increases from 5 to 20 m/s for a 16,000 t vessel. These values look low relative to AASHTO-type estimates and should be verified. — [Wiley 10.1155/stc/5568505](https://onlinelibrary.wiley.com/doi/10.1155/stc/5568505); [UH repository](https://uhra.herts.ac.uk/id/eprint/26849/)
- **RC pier damage evolution:** ML identification of RC pier damage evolution under vessel collision with multi-hazards. — [Ocean Eng. 2025 S0029801825001490](https://www.sciencedirect.com/science/article/abs/pii/S0029801825001490)
- **GP surrogates elsewhere:**
  - Fragility of RC bridge piers using OpenSees inputs. — [RG 363648808](https://www.researchgate.net/publication/363648808_Machine_Learning-based_Surrogate_Model_for_RC_Bridge_Pier_Collapse_Fragility_Prediction)
  - Train–track–bridge rock impact (2025), with UQ and Sobol sensitivity on top of the GPR surrogate. — [Springer s43251-025-00200-5](https://link.springer.com/article/10.1186/s43251-025-00200-5)
- **Active learning:**
  - PC-Kriging surrogates trained with a LOO-CV variance correction and U-function learning. The case study is ship collision with an offshore wind substructure (failure and repair events). — [arXiv 2302.12074](https://arxiv.org/pdf/2302.12074)
  - Active-learning PC-Kriging for marine structures: U-function plus K-means multi-point enrichment. — [RG 353062794](https://researchgate.net/publication/353062794_An_Active_Learning_Polynomial_Chaos_Kriging_metamodel_for_reliability_assessment_of_marine_structures)
  - Kriging with subset simulation, using a two-stage convergence criterion. — [RESS 2021](https://www.sciencedirect.com/science/article/abs/pii/S0951832021002702)
  - Multi-objective exploration/exploitation in active learning. — [arXiv 2508.18170](https://arxiv.org/pdf/2508.18170)
- **NSGA-II in maritime work:**
  - Collision-avoidance strategy trading safety against economy. — [Wiley 2014](https://onlinelibrary.wiley.com/doi/epdf/10.1155/2014/914689)
  - Multi-ship encounters in complex waterways. — [Ocean Eng. 2024](https://www.sciencedirect.com/science/article/abs/pii/S0029801824028506)
  - Multi-stage, multi-objective avoidance with risk preferences. — [Ocean Eng. 2023](https://www.sciencedirect.com/science/article/abs/pii/S0029801823022722)
  - SAR resource allocation in the South China Sea. — [JMSE 12:184](https://doi.org/10.3390/jmse12010184)
  - Ship-noise reduction, with TOPSIS to pick a Pareto solution. — [arXiv 2402.02647](https://arxiv.org/pdf/2402.02647)
- **Closest to bridge mitigation optimisation:** "Multi-objective optimisation of surveillance camera placement for bridge–ship collision early-warning using an improved non-dominated sorting genetic algorithm" (Advanced Engineering Informatics, 2025). Only the title was available. — [S1474034625008110](https://www.sciencedirect.com/science/article/abs/pii/S1474034625008110)
- **Optimal traffic width (single-parameter mitigation):** traffic width passing an OWF optimised from IWRAP collision probability. — [Ocean Eng. 2024](https://www.sciencedirect.com/science/article/abs/pii/S0029801824028361)
- **Active push-assisted yaw correction:** ESO + fuzzy PID control for bridge-area vessels (2026). — [Appl. Sci. 16:2520](https://doi.org/10.3390/app16052520)

### Inferences
- **Engine architecture:**
  - A physics engine computing frequency × P(failure | impact) can embed an active-learning PC-Kriging or GP surrogate of P(structural failure | ship mass, speed, angle, impact location, pontoon or anchor state), trained on a few hundred nonlinear FE runs.
  - That makes millions of Monte Carlo samples tractable and gives surrogate-error UQ, via the Kriging variance and Sobol indices.
- **RCO optimisation:**
  - **Inputs:** a risk-control-option (RCO) portfolio, for example fenders or dolphins, protective islands, speed limits, tug escort, VTS upgrade, an active warning system, AIS-based sensors or camera placement, and traffic-separation geometry.
  - **Objectives:** cost, residual risk (PLL or annual allision frequency), downtime and possibly emissions.
  - **Method:** NSGA-II over the fast surrogate engine, with TOPSIS or ALARP/CAF selection, or Bayesian optimisation where engine runs are expensive.
  - The algorithms are **M**; the application to bridge allision is **R/novel**, and therefore a differentiator.

### Gaps
- No published NSGA-II or Bayesian-optimisation study of ship–bridge allision RCO portfolios was found.
- Full-text accuracy metrics (R², error) for the 2023 barge–pier Kriging/NN framework and the 2026 Haq et al. framework were not retrievable.
- No surrogate studies for floating-bridge pontoon/girder impact or mooring response under ship impact were found in this search.

---

## 5. LLM / agentic AI in maritime and structural/risk engineering (2023–2026): simulation orchestration, report generation, standards RAG, maritime LLMs, MCP toolchains

### Takeaway
LLM agents that orchestrate verified solvers through tool calling or MCP are an emerging, research-grade pattern. Accuracy is high when the LLM only drives the solver (under 1.5% deviation), and unacceptable when it computes itself (over 400% deviation). General LLMs remain unreliable for COLREGs and situational reasoning. Building-code RAG works with simple retrievers. Maritime accident-report RAG and KG work is the most mature LLM use in this domain.

### Cited Findings
- **MCP + structural FE:** a Human–AI teaming preprint connects GPT-4 to OpenSeesPy through MCP. It shows under 1.5% deviation from reference solutions over four case studies, versus over 400% for unconstrained LLM computation. Its design principle is to delegate computation to verified solvers and use the LLM as the interface. Not peer-reviewed. — [preprints.org](https://www.preprints.org/frontend/manuscript/6b448577b50549815257006fae747503/download_pub)
- **Agentic FE frameworks:**
  - PDE-Agents: Simulation, Analytics and Database agents under a LangGraph supervisor, with GraphRAG (Neo4j plus 768-d embeddings), on local open models. — [arXiv 2606.07850](https://arxiv.org/html/2606.07850)
  - ALL-FEM: fine-tuned agentic LLMs for FEniCS, with 71.8% code-level success on 39 benchmarks (CMAME 2026). — [ScienceDirect S0045782526002586](https://www.sciencedirect.com/science/article/abs/pii/S0045782526002586)
  - VFEAgent: a multimodal end-to-end FEA agent. — [review](https://www.themoonlight.io/en/review/vfeagent-a-multimodal-agent-framework-for-end-to-end-automated-finite-element-analysis)
  - DUCTILE: agentic orchestration of engineering analysis. — [arXiv 2603.10249](https://arxiv.org/pdf/2603.10249)
  - Grid-Orch: 36 simulation capabilities exposed as MCP tools. — [arXiv 2605.12728](https://arxiv.org/pdf/2605.12728)
  - AVEVA process simulation via MCP; the authors say expert oversight is still needed. — [arXiv 2601.11650](https://arxiv.org/pdf/2601.11650)
  - Multi-agent HPC screening orchestration. — [arXiv 2604.07681](https://arxiv.org/pdf/2604.07681)
- **Standards RAG:**
  - Canada National Building Code QA: Elasticsearch was the best retriever, and top-3 to top-5 retrieved chunks were sufficient. — [arXiv 2505.04666](https://arxiv.org/pdf/2505.04666); [EurekAlert](https://www.eurekalert.org/news-releases/1098960)
  - IBC 2015 RAG beats the state of the art on a 150-record set. — [IAARC CSCE 2025](https://www.iaarc.org/publications/csce_crc_2025/an_llm_based_framework_with_retrieval_augmented_generation_for_building_code_interpretation.html)
  - Construction-management RAG: Top-3 retrieval 0.924, answer accuracy 0.898. — [Adv. Eng. Informatics 2025](https://www.sciencedirect.com/science/article/abs/pii/S1474034625000515)
- **Maritime LLMs:**
  - MarineGPT (2023) is a multimodal ocean-domain model. — [arXiv 2310.13596](https://arxiv.org/pdf/2310.13596)
  - LLaMarine is an open-source maritime-industry LLM. — [RG COLREG3 refs](https://www.researchgate.net/publication/378744986_COLREG3_-_Exploring_the_potential_of_Large_Language_Models_in_marine_navigation_systems)
  - COLREG3 (Sweden) found GPT-4 makes basic mistakes, such as misjudging a target ship's relative position. — [RG 378744986](https://www.researchgate.net/publication/378744986_COLREG3_-_Exploring_the_potential_of_Large_Language_Models_in_marine_navigation_systems)
  - A prompt-engineered GPT-4 decision system handled COLREG Rules 13, 14, 16 and 17 in test scenarios. — [arXiv 2411.16587](https://arxiv.org/pdf/2411.16587)
  - A 50-scenario Danish real-encounter benchmark evaluated 5 LLMs single-shot (2026). — [arXiv 2608.08281](https://arxiv.org/html/2608.08281v1)
  - A Missouri State study compared ChatGPT, DeepSeek and Claude on 5 multi-vessel scenarios. — [BearWorks](https://bearworks.missouristate.edu/articles00/193/)
  - Other benchmarks: IMDG Code compliance evaluation. — [arXiv 2608.21036](https://arxiv.org/pdf/2608.21036)
  - CyPortQA: multimodal port-cyclone benchmark. — [arXiv 2508.15846](https://arxiv.org/pdf/2508.15846)
  - Context-enriched natural-language descriptions of vessel trajectories. — [arXiv 2603.12287](https://arxiv.org/pdf/2603.12287)
  - A report on LLMs in maritime data analysis and decision support (Lighthouse, Sweden). — [Lighthouse](https://lighthouse.nu/images/Rapporter/Large_Language_Models_LLMs_in_Maritime_Data_Analysis_and_Decision_Support.pdf)
- **Accident-report LLM/RAG:** see Section 3 (ontology-constrained prompting with F1 about 94.7%, MaRAG, MAIB RAG with prompt-injection testing). The DLSCA framework combines deep learning with a LoRA-fine-tuned LLM and RAG for collision-accident analysis. — [search summary of LLM-maritime results](https://doi.org/10.3390/jmse14110983)

### Inferences
- **Product pattern** (R-grade but low technical risk if solvers stay authoritative):
  - Wrap the physics allision engine, the FE surrogate, AIS analytics and report templates as MCP tools.
  - Let an LLM agent orchestrate scenario runs, sensitivity sweeps and NSGA-II jobs.
  - Have it draft risk reports with citations into a RAG store of AASHTO, EN 1991-1-7, PIANC and IALA guidance.
  - Never let the LLM compute frequencies or capacities itself; the less than 1.5% versus over 400% contrast is the key evidence.
- **Not for warning logic:** LLMs should not sit in the real-time warning decision loop. COLREG and situational benchmarks show basic spatial-reasoning errors. They are suitable for operator explanation, incident summarisation and VTS log analysis.

### Gaps
- No peer-reviewed MCP-based structural or risk toolchain with field validation was found; the OpenSees MCP work is a preprint.
- No LLM evaluation was found on Eurocode, AASHTO LRFD vessel-collision provisions or PIANC bridge-protection guidance.
- No published LLM deployment inside an operational VTS was found.

---

## 6. Real-time ship–bridge collision warning systems using AI (China, Hong Kong, US post-Key Bridge), radar + AIS + video fusion, and AI-enhanced VTS: reported performance

### Takeaway
Active anti-collision warning (AIS + radar + CCTV/video + lidar, with YOLO-type detection, DeepSort tracking and trajectory prediction) is operational at many Chinese bridges. The Yangtze Wuhan reach went from its first system in 2020 to 7 bridges by 2023, and the "largest bridge cluster" on the middle and lower Yangtze reported full coverage in 2024. Peer-reviewed performance numbers are mostly component-level (detection mAP about 99%, size-measurement MAPE about 3.7%). No independent field statistics on false alarms, missed alarms or prevented allisions were found. Model studies credit active warning with a modest risk reduction compared with passive protection. In the US, post-Key Bridge activity is dominated by NTSB-driven vulnerability assessments; the only AI warning deployment found is a vendor claim.

### Cited Findings

**Peer-reviewed systems (China)**
- Baijusi Yangtze River Bridge warning system: YOLOv5 detection, DeepSort tracking, video–AIS association, and a ship-manoeuvrability model for collision potential. — [ASCE-ASME J. Risk Uncertainty A, RUENG-1590](https://ascelibrary.org/doi/10.1061/AJRUA6.RUENG-1590)
- Camera array + AIS fusion with confidence weighting, which keeps tracking through AIS dropouts and issues tiered warnings. — [Engineering Structures 2025 S0141029625022588](https://www.sciencedirect.com/science/article/abs/pii/S0141029625022588)
- A vessel–bridge and vessel–vessel warning framework (Comput.-Aided Civ. Infrastruct. Eng. 37(5):629–649, 2022) uses homography positioning, GAN trajectory augmentation and dual-task anomaly detection. — [Wiley mice.12757](https://onlinelibrary.wiley.com/doi/10.1111/mice.12757)
- Modified YOLOv5 for vessel–bridge warning in complex scenes: 99.4% mAP for crucial vessels at 102 FPS. — [J. Southeast Univ. 2024](https://journal.hep.com.cn/jseu/EN/10.3969/j.issn.1003-7985.2024.01.004)
- Digital-twin-assisted automatic ship-size measurement for over-height detection: trained on a synthetic 3D environment with transfer to real footage, MAPE about 3.74% across camera–ship distances. — [JZUS-A 2025](https://link.springer.com/article/10.1631/jzus.A2500277)
- AI multi-sensor proactive warning (Ni Liu, T&F chapter): camera, lidar, radar, AIS and weather sensors; YOLO combined with AIS/radar; image restoration for rain, fog and low light; and occlusion trajectory prediction. — [T&F 9781003778677-7](https://www.taylorfrancis.com/chapters/oa-edit/10.1201/9781003778677-7/proactive-bridge-anti-ship-collision-warning-system-using-ai-multi-sensor-fusion-ni-liu)
- Vision–radar fusion pre-warning decision method tested at Youdungang bridge, reported as having "fair risk identification ability". — [T&F 9781003483755-419](https://www.taylorfrancis.com/chapters/oa-edit/10.1201/9781003483755-419/decision-method-ship-bridge-collision-pre-warning-based-vision-radar-data-fusion-hongyu-zhuang-dalei-wang-zhenxiong-gu-yiqing-dong-junjie-wang)
- Tongji University (IABSE 2021): a bridge–vessel monitoring system on an NVIDIA edge board with sound and light alarms. — [structurae](https://structurae.de/de/fachliteratur/tagungsbeitrag/bridge-vessel-anti-collision-monitoring-system-design-and-implementation)
- A 2025 Chinese journal paper describes a YOLOv5 + phased-array radar system with three warning zones that trigger sound and light alarms or an interception device. — [napstic](https://search.napstic.cn/literature/periodical/010jxzz202505019)
- Patent CN113077659A covers multi-source-fusion ship–bridge warning based on predicted trajectories and collision probability. — [Google Patents](https://patents.google.com/patent/CN113077659A/zh)
- A camera-calibration and motion-detection bridge collision-avoidance system (Sci. Rep. 2025). It cites Thombre et al. (2020): radar + depth + RGB fusion with Bayesian filtering and Dempster–Shafer, 97% threat-detection precision. That precision figure is second-hand and comes from an autonomous-ship context. — [Sci. Rep. s41598-025-19096-2](https://www.nature.com/articles/s41598-025-19096-2)
- Ma et al. (Sensors 2022) note that active early-warning systems operate through VTS, AIS and CCTV video detection, and that false and missed alarms remain a problem. Modelled active-measure reduction: 2.2×10⁻³ → 1.6×10⁻³. — [Sensors 22:2857](https://doi.org/10.3390/s22082857)
- An active early-warning design for inland bridge piers uses IR and visible cameras, laser detection and a Kalman-filter predictor. The authors recommend combining it with passive systems. — [RG 288108545](https://www.researchgate.net/publication/288108545_Active_early-warning_system_for_bridge_piers_against_ship_collision_and_its_performance_analysis)

**Deployment reports (China; media and vendor sources)**
- Qingshan Yangtze River Bridge (2020) was the first in the Wuhan reach with an active anti-collision warning system. By 2023, 7 Wuhan bridges used radar, laser, video and AIS fusion. — [Sohu](https://www.sohu.com/a/1050280765_121667120); [QQ News 2024](https://news.qq.com/rain/a/20240304A05SYS00)
- Vendor and blogger sources:
  - AIS-only off-course judgement is unreliable because AIS can be switched off, is delayed, has 10 m-class accuracy and an unstable update rate, so false alarms are high.
  - Microwave radar fused with AIS is also said to have high false alarms.
  - Video is weather- and range-limited.
  - Multi-line lidar is promoted for over-height and off-course detection (for example at the Xiangjiang highway bridge).
  - These are non-authoritative and marketing in nature.
  - — [Zhihu 533897262](https://zhuanlan.zhihu.com/p/533897262); [Zhihu 609820062](https://zhuanlan.zhihu.com/p/609820062); [Seyond](https://www.seyond.cn/%E6%A1%A5%E6%A2%81%E4%B8%BB%E5%8A%A8%E9%98%B2%E6%92%9E%E9%A2%84%E8%AD%A6%EF%BC%81%E5%9B%BE%E8%BE%BE%E9%80%9A%E6%BF%80%E5%85%89%E9%9B%B7%E8%BE%BE%E4%B8%BA%E6%B9%98%E6%B1%9F%E5%85%AC%E8%B7%AF%E5%A4%A7/); [ZiDongHua](https://www.zidonghua.com.cn/news/program/60452.html)
- A Chinese national "three-year action plan for remediation of ship–bridge collision hazards" is referenced as a driver of active-warning procurement. — [Zhihu 489655726](https://zhuanlan.zhihu.com/p/489655726)

**Hong Kong / Hong Kong–Zhuhai–Macao Bridge (HZMB)**
- HZMB is described as having an AIS-based ship-collision early-warning system that outperforms traditional VTS. This is an encyclopaedia-level source. — [Baidu Baike EN](https://baike.baidu.com/en/item/Hong%20Kong-Zhuhai-Macao%20Bridge/980383)
- An SCMP article describes a Hong Kong (PolyU) invention combining AIS and vision to warn ships approaching bridges. — [SCMP](https://www.scmp.com/news/hong-kong/article/1233083/invention-prevents-ships-colliding-nearby-bridges)
- PolyU's technology listing adds embedded piezoelectric impact-force sensors. — [PolyU KTEO](https://www.polyu.edu.hk/kteo/knowledge-transfer/innovations-and-technologies/technology-search/2-construction-and-environmental-technology/2_cee_03_1113/)
- In 2016 a high-speed passenger craft struck HZMB anti-collision piles in poor visibility. The search summary attributes this to lack of lookout and unsafe speed, with 3 piles deformed. — [Baidu Baike EN](https://baike.baidu.com/en/item/Hong%20Kong-Zhuhai-Macao%20Bridge/980383)
- Tsing Ma Bridge has a ship-impact detection alarm (4 CCTV cameras at the towers) integrated into its SHM system. — [J. Civil SHM 2024](https://link.springer.com/article/10.1007/s13349-024-00842-5)

**US (post-Key Bridge)**
- NTSB March 2025: 68 bridges and 30 owners in 19 states were asked to run vulnerability assessments; MIR-25-10 calls for current vessel-traffic data. — [NTSB](https://www.ntsb.gov/news/press-releases/Pages/nr20250320.aspx); [MIR2510](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2510.pdf)
- Experts disagree on whether any protection could save a bridge from a Dali-size strike; they call for fenders and renewed assessments (CNN, May 2026). — [KESQ/CNN](https://kesq.com/news/national-world/cnn-national/2026/05/19/major-us-bridges-could-be-vulnerable-to-ship-collisions-including-one-just-downstream-from-key-bridge-2/)
- Roebling Labs (vendor, unverified) claims:
  - AIS threat levels from tracking, trajectory forecasting and AASHTO impact analysis;
  - a site near Boston's Tobin Bridge running since January 2026;
  - 45 bridges still on the NTSB list as of April 2026.

  A separate search could not corroborate any of this. — [Roebling Labs](https://www.roeblinglabs.com/)

**AI-enhanced VTS**
- Singapore MPA's Next-Generation VTMS can warn ships up to 30 min ahead about trouble spots (around 2019 article). OCEANS-X provides 100+ APIs and datasets for AI tools. — [Port Strategy](https://www.portstrategy.com/news/digital/targeting-efficiencies/)
- An April 2026 "AI-powered VTMS pilot" (port not clearly identified) claims "potential" for up to 15% reduction in waiting times. This is an efficiency metric, not a safety one. — [TechFinitive](https://www.techfinitive.com/features/how-the-worlds-largest-ports-are-using-ai-to-keep-the-global-supply-chain-humming/)
- Rotterdam's partnership with Awake.ai is for logistics ML, not VTS alarms. — [Port Strategy](https://www.portstrategy.com/news/digital/targeting-efficiencies/)
- AI real-time risk detection in ship navigation. — [Appl. Sci. 15:11674](https://www.mdpi.com/2076-3417/15/21/11674)
- Fuzzy-inference multi-vessel risk from the VTS viewpoint. — [ESWA 2012](https://www.sciencedirect.com/science/article/abs/pii/S0957417412009773)

### Inferences
- **State of the art for a warning product:** the SOTA reference architecture is:
  - AIS for identity and intent;
  - radar and/or lidar for non-AIS and AIS-dark targets;
  - CCTV with YOLO-class detection plus DeepSort for track continuity and classification;
  - image restoration for fog;
  - a manoeuvrability-aware prediction (Kalman/UKF or LSTM-UKF) of time-to-allision;
  - tiered alarms to the vessel (VHF/AIS safety message, sound and light) and to VTS or bridge operators.

  The components are **M**; the evidence that alarms cause risk reduction is weak.
- **How to credit warning in the engine:** credit active warning as a Pc or failure-to-avoid reduction factor conditioned on available reaction time and the ship's stopping/turning capability, as in the Baijusi manoeuvrability model. This matters most for powered course-deviation scenarios and little for drifting or blackout scenarios, which favours passive measures (consistent with Ma et al.).
- **KPIs to report:** a product should report false-alarm rate per day, missed-detection rate, median warning lead time and coverage under fog. These are exactly the metrics missing from the literature, and their absence is a credibility opportunity.

### Gaps
- No independent field statistics on false and missed alarms or prevented allisions from Chinese deployments (Wuhan, Nanjing, HZMB) were found. Authoritative MSA, Changjiang MSA and acceptance reports were not reachable.
- The current (2024–2026) HZMB warning-system architecture is not documented in authoritative English sources.
- No US AI-based bridge-allision warning pilot at the Key Bridge or another public owner was confirmed beyond the unverified vendor claim.
- The CACAIE 2022 GAN dual-task framework's quantitative results could not be retrieved.

---

## 7. Digital twins for bridges and waterways combining physics and ML

### Takeaway
Bridge digital twins combining FE, sensing and ML exist mostly as research demonstrators and pilots. Authors explicitly say hybrid frameworks with field validation are lacking. Waterway and vessel digital twins are mostly about autonomous surface vessels (RL plus NMPC). The only bridge–ship collision digital-twin paper found uses the twin for synthetic training data for ship-size measurement, not as a coupled risk twin. A coupled "traffic + ship hydrodynamics + structural + ML" twin for allision appears not to exist in the literature, which is a product opportunity, but research-grade.

### Cited Findings
- Digital-twin-assisted ship-size measurement for ship–bridge early warning: synthetic 3D training data, transfer learning, MAPE about 3.74%. — [JZUS-A 2025](https://link.springer.com/article/10.1631/jzus.A2500277)
- Melbourne/WSU multi-layer bridge DT (FE + drone photogrammetry + wireless sensors, 2025). The authors state that hybrid physics–data frameworks with operational field validation are still lacking. — [RG 397750772](https://www.researchgate.net/publication/397750772_A_Demonstration_of_a_Digital_Twin_Framework_for_Structural_Health_Monitoring_Application_to_Bridge_Infrastructures)
- Other bridge-twin research:
  - Traffic- and weather-driven hybrid DT that uses cameras, a physics traffic simulation, fatigue modelling and ML forecasting. — [arXiv 2603.14028](https://arxiv.org/html/2603.14028)
  - Data-driven DT with unsupervised deep learning on Werrington Bridge accelerations, needing no damage data. — [Information Fusion 2025 S1566253525006062](https://www.sciencedirect.com/science/article/pii/S1566253525006062)
  - Probabilistic DT calibration that embeds model-form uncertainty. — [arXiv 2609.10171](https://arxiv.org/pdf/2609.10171)
  - Physics-guided graph Kalman filters for virtual sensing. — [arXiv 2604.26593](https://arxiv.org/pdf/2604.26593)
  - Active transfer learning for SHM. — [arXiv 2510.27525](https://arxiv.org/pdf/2510.27525)
  - Bridge-management digitalisation framework. — [ASCE JITSE](https://ascelibrary.org/doi/10.1061/JITSE4.ISENG-2757)
- Sea-crossing infrastructure SHM:
  - HZMB bridge–tunnel SHM system design. — [Shock Vib. 2019](https://onlinelibrary.wiley.com/doi/10.1155/2019/2832089)
  - Over 25 years of Tsing Ma monitoring, including a ship-impact alarm. — [J. Civil SHM 2024](https://link.springer.com/article/10.1007/s13349-024-00842-5)
- Maritime digital twins:
  - A review notes that physics-based twins drift out of sync and are simplified for computation, and that ML can close the gap. — [ai4europe nav2022](https://www.ai4europe.eu/sites/default/files/2023-12/nav2022-v0.4.pdf)
  - ASV twins with RL plus NMPC and physics environment models. — [Sci. Rep. 2025](https://www.nature.com/articles/s41598-025-93635-9); [arXiv 2401.04032](https://arxiv.org/pdf/2401.04032)
  - Purely data-driven twins lack interpretability. — [arXiv 2411.03465](https://arxiv.org/pdf/2411.03465)
  - Marine-operations DT review. — [arXiv 2301.09574](https://arxiv.org/pdf/2301.09574)
- Drift physics + ML hybrid (ConvLSTM environmental forcing). — [Ocean Eng. 2025](https://www.sciencedirect.com/science/article/abs/pii/S0029801825032421)

### Inferences

**Proposed hybrid allision twin**

| Layer | Content | Maturity |
|---|---|---|
| (a) Traffic | AIS ingestion and cleaning (AISdb-type) → route graph → KDE/GMM lateral distributions → ship-class mix | M |
| (b) Failure/causation | IWRAP-type geometric × Pc, with a BN Pc-modifier layer, Bayesian-updated by AIS near-allision detections | P |
| (c) Drift | OpenDrift-type physics with metocean forecasts, optionally an ML-corrected forcing | P / R |
| (d) Consequence | Active-learning PC-Kriging/GP surrogate of nonlinear FE (pontoon, girder, anchors), with UQ | P |
| (e) Operational mode | Live fusion of AIS, radar, video and lidar with probabilistic trajectory prediction → time-to-allision alarms; the live state also updates the risk dashboard | M components, R for validated alarm performance |
| (f) Optimisation | NSGA-II / Bayesian optimisation over the RCO portfolio using the surrogate engine | R application |
| (g) Agent layer | LLM over MCP orchestrating runs, RAG over standards, report drafting and accident-report mining | R |

**Maturity matrix**

| Method | Maturity | Representative refs (year) | Data needs | Reported accuracy |
|---|---|---|---|---|
| AIS cleaning/segmentation (AISdb, α-method) | M | 2024 | Raw AIS (terrestrial or satellite) | n/a |
| Density route extraction (DBSCAN/OPTICS/HDBSCAN, TREAD, graph) | M/P | 2013–2025 | Months–years of AIS | <5% outliers (2025) |
| KDE/GMM lateral distributions | M/P | 2018–2024 | AIS per cross-section | n/a |
| Ship-type classification | P | 2024–2026 | AIS static + dynamic | 95–97% (4–5 classes); about 78% (14 classes) |
| Deep trajectory prediction (LSTM/Transformer/GNN) | P short-term / R for safety use | 2021–2026 | Large AIS corpora | TrAISformer <10 nmi @ about 10 h; 1-h error 2.2→24.6 km out of region |
| LLM trajectory/COLREG reasoning | R | 2023–2026 | AIS + text | Basic spatial errors |
| AIS anomaly detection (generic) | P | 2020–2026 | AIS | Varies; no standard benchmark |
| Loss-of-propulsion detection from AIS | R / not found | — | AIS + metocean | — |
| Ship-domain / CPA near-miss detection | M/P | 2015–2026 | AIS | 19/47 confirmed by manual review (Hassel 2020) |
| EVT near-miss → frequency | R | 2026 | AIS | Huangpu 0.21%, 11.54/yr |
| Data-driven BN on accident databases | P | 2018–2026 | Investigation reports | >93% consequence accuracy (TAN 2026) |
| Spatial ML accident prediction | P | 2022–2024 | Accidents + AIS + environment | AUC >0.9 (Rawson 2022) |
| LLM/NLP accident-report knowledge graphs | P/R | 2024–2026 | Reports | F1 0.87–0.95 |
| FE impact surrogates (Kriging/NN/RF) | P | 2009–2026 | 100s of FE runs | Not retrievable |
| Active-learning PC-Kriging reliability | M (reliability field) | 2021–2023 | FE runs | — |
| NSGA-II / Bayesian optimisation of RCO portfolio | R (white space) | none found | Engine + cost data | — |
| MCP/LLM solver orchestration | R | 2025–2026 preprints | Tool APIs | <1.5% deviation vs >400% unconstrained |
| Multi-sensor bridge warning (YOLO + DeepSort + AIS + radar) | M (China deployments) | 2019–2025 | Sensors at site | mAP 99.4%; size MAPE 3.74%; no field false-alarm stats |
| Hybrid bridge/waterway DT | R | 2025–2026 | SHM + AIS + metocean | — |

### Gaps
- No published coupled traffic–drift–structural–ML digital twin for ship–bridge allision was found, and none specifically for floating bridges.
- No floating-bridge (pontoon) ship-impact surrogate or digital-twin literature was retrieved in this search.
- Full-text access was blocked for all major publishers, so maturity judgements rely on abstracts and snippets. The report writer should treat author-reported accuracies as indicative only.
