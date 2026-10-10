# Kamzela et al. 2026: frozen Aurora plus regional heads

Kamzela, Kubiak, Dobosz, Miczke, Kaczmarek, Wyrwiński, Stefaniak, Kotłowski (Poznań Supercomputing and Networking Center; Poznań University of Technology). *From Global to Local: Efficient Regional Weather Downscaling with Global Weather Foundation Model.* arXiv [2607.03279](https://arxiv.org/abs/2607.03279)v1, 3 July 2026, preprint. No code release is mentioned. Summary checked against the paper on 1 October 2026.

**In short.**
- **Method.** Freeze Aurora 0.25°, cache its 1024-d embedding for each 1°×1° patch, and train small MLP heads that turn each embedding into a 40×40 patch at 0.025° (≈2.2 km). The heads learn from regional model fields over the continental US.
- **Result.** At +6 h, scored at ~1,800 stations, the best head beats HRRR's own WRF forecast and plain Aurora. Reading the embedding beats refining Aurora's output.
- **Limits.** It's deterministic and tested mostly at +6 h from ERA5. For 2 m temperature at stations it only just beats interpolated ERA5.

## What they did

- **Backbone:** Aurora 0.25° (MIT licence), frozen; ERA5 in. Aurora decodes each 1° patch from its own embedding, so heads can be bolted on patch by patch.
- **Heads:**
  - MLPs with 0–2 hidden layers of width 1024 map an embedding to a 40×40 patch.
  - The best model sums three parts: a fine head (40×40), a coarse head (8×8, upsampled), and Aurora's own linear decoder.
  - The fine head also gets a *learned* encoding of the patch's 40×40 high-resolution elevation.
- **Training targets:** model fields, not stations.
  - WTK-US: WRF at ~2 km, 2007–2013.
  - HRRR analyses: ~3 km, 2017–2024, with 2022 held out.
  - The best model trains on both. Variables are 2 m temperature, 10 m wind and surface pressure, 6-hourly, with an MAE loss.
- **Cost:**
  - Backbone embeddings are cached once (~350 GB); the whole corpus, targets included, is ~30 TB.
  - A head trains in ~10 h on one H100 with ~20 GB of GPU memory.
  - The cost of building the cache (global Aurora passes) is not reported.
- **Evaluation:**
  - +6 h forecasts from ERA5, scored on the HRRR grid (2022) and at ~1,800 CONUS HadISD stations (nearest 0.025° cell).
  - RMSE in the tables; station MAE maps and energy spectra in the appendix.
  - A 48 h rollout is compared with HRRR's WRF-ARW forecasts.

## Key numbers: 2 m temperature RMSE (°C), +6 h

| Model | Stations | HRRR grid |
|---|---|---|
| Best head (two scales, elevation encoder) | **1.70** | 1.25 |
| Same, no elevation encoder | 1.74 | 1.29 |
| WRF-ARW (HRRR's forecast) | 1.88 | **1.17** |
| Aurora's decoder, fine-tuned on HRRR | 2.01 | 1.64 |
| Aurora 0.25° (Table 3) | 2.05 | 1.86 |
| ERA5, bicubic (valid time not stated, probably the reanalysis) | 1.72 | 2.35 |
| StormCast, 5 members | 2.43 | 2.24 |

- **Latent vs output.** A head reading the embedding (summed with Aurora's own output) beats a head refining only Aurora's 4×4 output. At stations: 1.973 vs 1.982 with linear heads (0.5 %), 1.857 vs 2.001 with one hidden layer (7 %); deeper was not compared. The gap is far larger for wind and pressure (grid pressure 0.93 vs 3.00 hPa).
- **Gains shrink from grid to stations.**
  - Against Aurora: 33 % better on the grid, 17 % at stations.
  - Against interpolated ERA5 at stations: 1.4 % for temperature, but 13–16 % for wind.
  - The station gains concentrate in mountains, on the California coast and around the Great Lakes.
- **Design choices.**
  - Depth helps, with diminishing returns.
  - A second, coarse head helps only once heads have hidden layers.
  - The learned elevation encoder helps temperature and wind but slightly hurts pressure. Raw elevation does nothing.
- **48 h rollout.** The text says error grows 18.5 % for the head vs 75.5 % for WRF-ARW, averaged over variables. For temperature, Fig. 3 reads as roughly +30 % vs +66 %. The head is slightly worse at +6 h and better from +12 h.
- **Other.**
  - One head trained on the US plus Denmark (DANRA) loses ~0.02 °C in each region.
  - Error falls along a smooth power law with more training data.

## Trust less

- **Tables disagree.** Table 6 doesn't match Tables 1 and 3 for the same models (e.g. 1.708 vs 1.743 at stations). The multi-region table (Table 5) uses an unstated setup.
- **Aurora baseline described two ways:** fine-tuned in one place, original in another.
- **Rollout percentages** can't be reproduced from Fig. 3.
- **Unstated:** how leads beyond +6 h are produced, and which Aurora latent level the heads read.

## Relevance to aifs-point-head

- **A second precedent after WN3,** which remains the nearest ([wn3-method.md](wn3-method.md)). What it adds: an open backbone from a third party, kept frozen, with heads trained on cached latents. Unlike WN3 and [idea.md](idea.md), it supervises with gridded model fields and uses stations only to score.
- **Its latent-vs-output result is against a weak output baseline:** Aurora's 4×4 patch, with no elevation and no neighbours. It doesn't settle our key control: the same head fed AIFS ENS output fields, plus a regional EMOS.
- **Which layer to read.** Aurora's patch embedding plays two roles that AIFS splits. By position (backbone output, ~1°, 1024 channels) it matches AIFS's O96 hidden latent. By role (what the final linear read-out reads) it matches the N320 decoder layer, which AIFS already builds by graph attention from the 3 nearest O96 nodes. So there are two options: read the N320 layer, or train a new decoder from the nearest hidden nodes, as GraphDOP and FMI's frozen-backbone observation decoder (anemoi-core issue #1400) do. The paper doesn't decide between them.
- **A two-stage regional head: plausible, untested here.**
  1. Train the head on the AIFS ENS latent against a dense regional analysis over NW Russia and Finland, which gives far more (correlated) targets than ~250 stations.
  2. Calibrate or fine-tune it on SYNOP at the training stations.

  Compare it with a station-only head and with the output-field control. The paper gives a reason to expect stage 2 matters: its grid gains shrink at stations (33 % → 17 % over Aurora; temperature level with interpolated ERA5).
- **Regional analyses covering Leningrad Oblast** (coverage checked 1 October 2026):

  | Dataset | Resolution and period | Covers |
  |---|---|---|
  | CERRA | 5.5 km; 3-hourly analyses, hourly short forecasts, a max-temperature parameter; from 1984, extended monthly with a 2–3 month lag | the whole oblast and the wider ~250-station region |
  | MEPS archive | 2.5 km; operational forecasts from Nov 2016 (domain changed Feb 2020), so inhomogeneous over time | the whole oblast, ≥100 grid cells from the edge |
  | MET Nordic analysis | 1 km hourly, from 2018, anchored to observations | St Petersburg and the west of the oblast; not the east (Tikhvin is outside) |
  | DANRA | 2.5 km | doesn't reach the oblast |

  An analysis that assimilated a held-out station carries that station's information into training. Check what each one assimilates before using it near held-out stations.
- **What the paper doesn't cover:** probabilities, dew point, and intermediate hours.
  - It is deterministic and predicts no humidity. Our plan keeps WN3's route: a head run per member and trained with fair CRPS, with dew point predicted alongside temperature.
  - Heads only output at 6-hourly valid times.
- **Lead time.** It mostly tests +6 h, and by its Fig. 3 its temperature error grows ~30 % by 48 h. Our scoring is per lead, so expect the same pattern.
- **Cost.** Heads are cheap. The latents need a backbone pass per time step, which our AIFS ENS reruns already do. Saving a regional patch of the latent, not single station points, keeps a gridded or regional head possible.
- **Worth copying:**
  - the learned elevation encoder;
  - the latent cache;
  - training one head across regions (weak evidence).
  - Fine and coarse heads apply only if the head outputs a grid rather than points.
