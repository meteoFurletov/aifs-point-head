# Reading guide: a head on frozen AIFS ENS

Short summaries to skim, each pointing at what it means for the experiment in [idea.md](idea.md). Every summary was written from the paper and then checked against it by a second reader (1 October 2026). **Read in full** marks the ones worth your whole time.

Covered separately: WeatherNext 3 ([wn3-method.md](wn3-method.md)) and Kamzela et al. 2026 ([kamzela-2026.md](kamzela-2026.md)).

## Start here

1. **AIFS-CRPS** (Lang et al. 2024): the backbone itself. Which layer to read, and where member spread comes from.
2. **Rasp & Lerch 2018:** how to build the control that feeds the same head AIFS ENS output fields instead of the internal state.
3. **Lakatos 2026:** the same unseen-station question on ECMWF 2 m temperature, with the controls we need.
4. **Milà et al. 2022:** how to build spatial folds that make the held-out score honest.
5. **Lehmann et al. 2025:** light decoders on a frozen model, and where they fail.
6. **Sousa et al. 2026:** which geography inputs carry local detail, with a control that catches false gains.
7. **Gneiting et al. 2025:** a clean test of whether the internal state holds more information than the output fields.

---

## Heads and decoders on frozen weather models

### AIFS-CRPS: the AIFS ENS backbone. Lang et al. 2024 · **Read in full**
[arXiv 2412.15832](https://arxiv.org/abs/2412.15832)
**Idea.** ECMWF's research version of AIFS ENS. Graph encoder and decoder, a 16-layer transformer processor (1024 channels, 229M parameters); members differ through noise injected into the processor's layer norms; trained with an "almost fair" CRPS.
**Key result.** 5–20 % better than the 9 km IFS ensemble on most upper-air scores (Feb–Sep 2024). One 15-day member takes ~4 min on an A100.
**For us.**
- Each step adds the decoder's output to the previous state smoothed to ~2.5°. So the ~31 km decoder layer carries fine detail and the 6 h change, not the whole state: give the head AIFS ENS 2t/2d too.
- For a 2-member loss, use its almost-fair CRPS (α = 0.95).

### Light decoders on frozen Aurora. Lehmann et al. 2025 · **Read in full**
[arXiv 2506.19088](https://arxiv.org/abs/2506.19088)
**Idea.** Aurora stays frozen; a small MLP per new variable (precipitation, evaporation, runoff, soil moisture) reads the 1° processor latent, trained on gridded targets. Compared with fine-tuning the whole model.
**Key result.** About twice as fast to train, with 65 vs 99 GB of GPU memory. Evaporation correlation 0.958 vs 0.992 fine-tuned; runoff only 0.42 vs 0.56.
**For us.**
- A 3-layer MLP on stored point features is a cheap first head.
- Where it failed (blocky 1° output, snowmelt runoff) warns that a ~1° latent blurs lake and coast contrasts unless geography inputs or the ~31 km layer supply them.
- No output-field control here.

### GraphDOP. Alexe et al. 2024 · Skim §3 and §5.1
[arXiv 2412.15687](https://arxiv.org/abs/2412.15687)
**Idea.** ECMWF's AIFS-like model trained only on observations. Its decoder links any point to its 3 nearest ~1° latent nodes, using bearing and distance, and predicts there.
**Key result.** Day-1 2 m temperature error 15 % below operational IFS at SYNOP stations (winter 2022–23; GraphDOP had ~5 h more data).
**For us.** A template for reading the ~1° processor latent at stations instead of bilinear interpolation. Add station-minus-model elevation, which their decoder lacked.

### A weather foundation model for the power grid. Bodnar et al. 2025 · Reference only
[arXiv 2509.25268](https://arxiv.org/abs/2509.25268)
**Idea.** A 1.5B-parameter model with a decoder that predicts at any point, fully post-trained (not frozen) on Hydro-Québec stations.
**Key result.** Temperature error ~15 % below IFS over 6–72 h at 534 stations.
**For us.** It shows the gaps to avoid: no held-out stations and no EMOS baseline. Break skill down by hour of day and season; winter was weakest.

## The baseline the head must beat

### Neural networks for post-processing ensembles. Rasp & Lerch 2018 · **Read in full**
[arXiv 1805.09091](https://arxiv.org/abs/1805.09091)
**Idea.** A small network maps the ensemble mean and spread of many fields, plus station information, to a Gaussian, trained on CRPS. It beats EMOS and quantile forests for 2 m temperature.
**Key result.** 48 h ECMWF ENS at ~500 German stations, CRPS: raw 1.16, regional EMOS 1.01, per-station EMOS 0.90, best network 0.78–0.82. Without station IDs: 0.86–0.90.
**For us.**
- Build the key control as this network fed AIFS ENS output fields, with the same geography inputs and loss as the head.
- Only the variants without station IDs can score held-out stations.

### EUPPBench. Demaeyer et al. 2023 · Skim
[ESSD 15:2635](https://essd.copernicus.org/articles/15/2635/2023/)
**Idea.** An open benchmark: IFS ENS 2 m temperature at 234 central-European stations, a shared protocol, and eight post-processing methods.
**Key result.** Up to 50 % lower CRPS than raw ENS with a height correction at early leads, 10–40 % at day 5.
**For us.**
- Copy the protocol: raw plus a 6.5 °C/km height correction as the reference, scored by lead and elevation. Reuse their published baseline code.
- ANET uses station metadata, not IDs, so it could work at unseen stations; that was never tested.

### IMPROVER on AIFS. Trotta et al. 2025 · Skim
[arXiv 2504.12672](https://arxiv.org/abs/2504.12672)
**Idea.** Australia's BoM ran its operational IMPROVER chain unchanged on deterministic AIFS, HRES and ENS for temperature, dew point and wind at stations.
**Key result.** Day-0 temperature CRPS after post-processing: AIFS 0.88, ENS 0.90, HRES 0.87 (569 stations). AIFS pulls ahead at later leads.
**For us.**
- Correcting toward a gridded analysis left biases at stations: a caution for training a head on CERRA without a final step on station data.
- Fit baselines on the same rerun years as the head (they had only 30 days of AIFS).

### AIFS ENS vs IFS ENS at stations (wind). Kocsis & Baran 2026 · Reference only
[arXiv 2606.02508](https://arxiv.org/abs/2606.02508)
**Idea.** Operational AIFS ENS v1 vs IFS ENS for 10 m wind at ~9,000 SYNOP stations, raw and after EMOS.
**Key result.** Raw AIFS ENS is ~4 % worse in CRPS than raw IFS ENS; EMOS closes most of the gap.
**For us.**
- Raw AIFS ENS is a weak bar at stations, so also report the head against EMOS on the same reruns.
- Their EMOS recipe (per lead, 60-day window, fitted on CRPS) carries over with a Gaussian.

### Pangu at Norwegian stations. Bremnes et al. 2024 · Skim
[NPG 31:247](https://npg.copernicus.org/articles/31/247/2024/)
**Idea.** Pangu run from ERA5 against HRES, ENS and the 2.5 km MEPS for temperature and wind at 183 Norwegian stations, raw and after quantile-network post-processing.
**Key result.** Post-processing cut 2 m temperature CRPS by up to ~50 % for the ECMWF models and Pangu; MEPS stayed best to at least 42 h.
**For us.**
- The nearest setup to ours: an AI model started from ERA5, scored at Nordic coast and valley stations.
- Their quantile network is a non-Gaussian option for dew point, once its station-ID embedding is swapped for metadata.

## Generalising to unseen stations

### Nearest-neighbour distance matching CV. Milà et al. 2022 · **Read in full**
[Methods Ecol. Evol. 13:1304](https://doi.org/10.1111/2041-210X.13851)
**Idea.** Cross-validation is honest only if held-out points sit as far from training points as the places you'll predict. NNDM drops training points in each fold until the two distance distributions match.
**Key result.** With clustered samples, plain leave-one-out underestimated error by 1.51 (in RMSE units); NNDM was off by only −0.03.
**For us.**
- Compare grid-to-station and station-to-station distances over the region. Build folds with the k-fold version (kNNDM) so held-out stations resemble station-free points.
- Plot held-out skill against distance to the nearest training station.

### Post-processing at unseen stations. Lakatos 2026 · **Read in full**
[arXiv 2609.07512](https://arxiv.org/abs/2609.07512)
**Idea.** 48 h ECMWF ENS 2 m temperature at 99 German stations with 30 % held out. Regional and per-station EMOS against a network, a transformer and a graph model.
**Key result.** CRPS at unseen stations: raw 1.10, network 0.874, regional EMOS 0.889. Per-station EMOS degrades from 0.78 at training stations to 1.07 at new ones.
**For us.**
- The same question as ours. Use regional EMOS and an output-field network without station IDs as controls.
- Report training and held-out scores side by side, and held-out scores by site type (elevation, coast, lake, city).

### Graph networks for post-processing. Feik, Lerch & Stühmer 2024 · Reference only
[arXiv 2407.11050](https://arxiv.org/abs/2407.11050)
**Idea.** Graph attention over station-and-member nodes, pooled to a Gaussian, at EUPPBench stations.
**Key result.** 1–2 % better than the network baseline at training stations; a version with no graph edges gets nearly all of that.
**For us.** No evidence that neighbours help at unseen stations. Keep station-ID embeddings out of the head.

## Local detail, humidity and intermediate hours

### Satellite embeddings as site descriptors. Sousa et al. 2026 · **Read in full**
[arXiv 2608.12271](https://arxiv.org/abs/2608.12271)
**Idea.** A model that downscales coarse fields to any point adds a compact summary of Sentinel-1/2 embeddings around each station as a description of the site. Trained on GHCNh stations.
**Key result.** 11.5 % lower 2 m temperature CRPS than elevation alone (hand-made land-cover features: 3.2 %). But shuffling the descriptors across stations keeps ~¾ of that gain.
**For us.**
- Test geography inputs as a ladder: elevation, then land-cover features, then satellite embeddings.
- Run each with a shuffle control. Check the embeddings cover Leningrad Oblast.

### Surface descriptors in a frozen Anemoi model. Bakketun et al. 2026 · Skim
[arXiv 2607.02824](https://arxiv.org/abs/2607.02824)
**Idea.** MET Norway freezes its Anemoi model and trains only a new decoder from the processor latent to 2.5 km. Surface descriptors (town, water, forest, terrain) enter only that decoder.
**Key result.** 2 m temperature error −1.9 % overall, −11.8 % over towns, −6.5 % on coasts (against the MEPS analysis, at +6 h).
**For us.**
- The closest frozen-Anemoi precedent. A decoder from the AIFS ENS ~1° latent straight to station points is the direct analogue.
- Report urban, coastal and lakeside stations separately.

### Physics-constrained temperature and humidity. Zanetta et al. 2023 · Skim §2d
[arXiv 2212.04487](https://arxiv.org/abs/2212.04487)
**Idea.** Predict temperature and a non-negative dew-point deficit, then derive dew point and humidity, so dew point never exceeds temperature.
**Key result.** No cost in accuracy (temperature error 1.340 vs 1.355 °C) and zero violations.
**For us.** Have the head output temperature and a deficit per member (Td = T − deficit) before scoring.

### HourGlass: hourly values from 6-hourly states. Ingstad et al. 2026 · Skim
[arXiv 2607.11457](https://arxiv.org/abs/2607.11457)
**Idea.** An Anemoi model turns two 6-hourly states into hourly values inside the window, keeping each member coherent. Trained with CRPS, including on the window's maximum and minimum.
**Key result.** Beats cubic splines for 2 m temperature and wind in the Nordic test; competitive with MEPS on the 24 h maximum.
**For us.**
- The baseline for intermediate hours (03/09/15/21 UTC).
- Test feeding the head both bounding states rather than one state plus time within the step.

### FMI's operational ML correction. Hieta & Partio 2025 · Reference only
[Met. Apps doi 10.1002/met.70074](https://doi.org/10.1002/met.70074)
**Idea.** XGBoost corrects FMI's hourly nowcast model at ~800–900 SYNOP stations from model output, station geography, time and the latest observed error.
**Key result.** Error 24–29 % below raw for leads 1–11 h, scored at the training stations.
**For us.**
- Its open code (fmidev/snwc_bc, MIT) is a cheap output-field control, minus the latest-error feature.
- It includes a seasonal SYNOP quality check for Finnish stations.

## Operational post-processing and fair evaluation

### Potential CRPS. Gneiting et al. 2025 · **Read in full**
[arXiv 2506.03744](https://arxiv.org/abs/2506.03744)
**Idea.** Pass every model's output through the same flexible calibration fitted on the evaluation data. The resulting "potential CRPS" measures the information in the output, whatever its bias.
**Key result.** The starting analysis alone (ERA5 or IFS) can reorder 2 m temperature rankings at short leads.
**For us.**
- An information test for the key comparison: compute it for raw AIFS ENS, the output-field head and the internal-state head. A gap means the internal state holds more than a recalibration.
- Rank only runs started from the same analysis.

### IMPROVER. Roberts et al. 2023 · Skim
[BAMS doi 10.1175/BAMS-D-21-0273.1](https://journals.ametsoc.org/view/journals/bams/104/3/BAMS-D-21-0273.1.xml)
**Idea.** The Met Office's probabilistic post-processing chain. Site temperature = careful choice of grid point + height correction from a local lapse rate + EMOS with the height difference as a predictor.
**Key result.** No aggregate scores; operational since 2022.
**For us.** The recipe for the regional EMOS control, with coefficients pooled over the region so it works at held-out stations. (The journal site blocks automated downloads; open it in a browser.)

### WeatherReal. Jin et al. 2024 · Skim §3
[arXiv 2409.09371](https://arxiv.org/abs/2409.09371)
**Idea.** Microsoft's station benchmark, with open quality control for NOAA ISD. It scores Aurora, IFS, GFS and a station post-processor.
**Key result.** 2023 2 m temperature error at stations: post-processor 2.26 K, Aurora 2.42 K, IFS 2.77 K.
**For us.**
- Reuse its ISD quality control for the 2015–24 truth, but inspect what its ERA5-based checks remove.
- Score 00 and 12 UTC separately.

### Site-specific temperature and dew point with XGBoost. Han et al. 2024 · Reference only
[arXiv 2404.03310](https://arxiv.org/abs/2404.03310)
**Idea.** A per-site XGBoost corrects IMPROVER's temperature and dew point toward station reports, using the site's grid point and its 8 neighbours.
**Key result.** Error −11 % for temperature and −12 % for dew point at 11 sites. Dropping the neighbouring points cost 0.26 °C.
**For us.** Test storing a small stencil of neighbouring nodes rather than the station point alone. Give the output-field control the same stencil.

## Alternative routes and failure modes

### Aardvark. Vaughan et al. 2024 · Skim
[Nature 2025](https://www.nature.com/articles/s41586-025-08897-0)
**Idea.** Observations in, a gridded state, then a station decoder for each lead (a UNet, kernel interpolation to points, then an elevation MLP). No NWP at forecast time.
**Key result.** Station 2 m temperature about as good as HRES corrected per station. Fine-tuning the whole chain added 3–6 %.
**For us.**
- Kernel interpolation with a learned length scale is an alternative to bilinear sampling of the latent.
- Its stations weren't held out by location; ours will be.

### AIFS-DOP. Pinnington et al. 2026 · Reference only
[arXiv 2606.19093](https://arxiv.org/abs/2606.19093)
**Idea.** The AIFS architecture trained only on observations gridded to ~1°, including SYNOP temperature and dew point.
**Key result.** Northern Hemisphere day-1 2 m temperature error ~2.4 vs 2.6 K for IFS, the gain all in winter.
**For us.**
- A possible later backbone whose latent already targets station temperature, but no weights are released.
- Score winter separately.

### Do AI weather models miss extremes? Gabler et al. 2026 · Skim
[arXiv 2608.09972](https://arxiv.org/abs/2608.09972)
**Idea.** Eleven systems scored at European stations by regime, with cold and hot tails taken from each station's climatology.
**Key result.** Deterministic AIFS loses 27 % to IFS in the severe-cold tail after bias removal. At 48 h every model is 0.7–1.2 °C too warm in severe cold.
**For us.**
- Score the head by regime, keeping the cold tail separate (NW Russia winters).
- Compare head and raw within the same groups, and group by forecast value too.
