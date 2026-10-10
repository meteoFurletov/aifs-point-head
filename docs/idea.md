# The idea: WN3's head on AIFS ENS

What this repo builds, and where it has to differ from WN3. The method itself is in [wn3-method.md](wn3-method.md); names and facts are in [estate.md](estate.md); related papers are in [reading-guide.md](reading-guide.md).

## Why

AIFS ENS 2 m temperature is a grid-box value at ~31 km and misses what a thermometer at a given spot reads. WN3 shows one way to close that gap: let a model trained on decades of analyses carry the representation, keep it frozen, and fit only a small head to station reports. AIFS ENS is open (weights CC BY 4.0, code Apache 2.0), was pre-trained on decades of ERA5 and fine-tuned on operational analyses 2016–2023, so it can be that frozen model.

The operational AIFS ENS archive starts only in July 2025, but our ERA5 reruns give ten years of its output fields as well as its internal state. So a correction learned from output fields is the bar to clear, not a dead end. The question this repo tests is whether the internal state carries station-scale information beyond the output fields: the head is compared with the same head fed output fields, and with a regional EMOS, on the same reruns.

## WN3 against this repo

| | WN3 | aifs-point-head |
|---|---|---|
| Backbone | WN3's own mesh transformer, trained by DeepMind | AIFS ENS v1, used as is, never trained |
| What the head reads | the decoded 0.1° latent, plus a skip from the inputs | AIFS ENS's internal state (the ~31 km decoder layer or the ~1° processor latent), plus its output fields |
| Metadata | station elevation, land or ship, time in the step (0.05° grid-box means at inference) | geography that varies between land stations: height above model orography, water and built-up fractions, distance to water |
| Outputs | 2 m temperature, 2 m dew point | the same, with dew point kept at or below temperature |
| Stations | METAR, Mesonet, ICOADS, global, ~20k | SYNOP and ISD: 13 in Leningrad Oblast, ~250 in 55.5–64°N, 20–42°E |
| Held out | a fixed random 5 % of stations, and later years | spatial folds of stations, scored after the backbone's training years |
| Pseudo-stations | 2,000 an hour from ERA5/HRES | none at first; judged by bias away from stations (over water, between stations), which held-out station skill cannot show |
| Loss | fair CRPS, 2 members | the same |
| Ensemble | from the backbone | from AIFS ENS's members |

## What has to be different

- **We run AIFS ENS ourselves.** ECMWF publishes output fields, not internal state, so the head's inputs come from our own reruns, started from ERA5. Both members start from the same unperturbed ERA5 state, as in AIFS ENS's own training, so "raw AIFS ENS" here means these reruns, not ECMWF's operational product, which starts from perturbed states. Raw, the output-field control and the head are all scored on the same reruns.
- **A coarser internal grid.** AIFS ENS computes on a ~1° grid; only its decoder's last layer is at ~31 km, against WN3's 0.1°. Local detail (lakes, coast, city) has to come from geography inputs; this is the biggest risk.
- **Few stations.** 13 in the oblast; about 250 if training covers 55.5–64°N, 20–42°E, nearly half of them Finnish. Spatial generalisation is the thing to test, not assume.
- **One AIFS ENS version throughout: v1**, the only one ERA5 can start ([estate.md](estate.md)). v1 has no snow depth, soil moisture, sea-surface temperature or sea ice among its inputs, so snow and lake ice reach the head only through surface and soil temperatures.
- **6-hourly steps.** WN3's backbone predicts hourly sub-steps; AIFS ENS does not. Reports between steps (03, 09, 15, 21 UTC) need both bounding states, or are scored apart.
- **In-sample years.** The backbone saw 2016–2023 in fine-tuning, so most of the head's training years are in-sample for it and the scoring years are not. Validation holds out years as well as stations.

## Data

- **ERA5 starting states**, streamed and never stored: each start reads two global states (the 6 h before and now) from ARCO-ERA5 on Google Cloud, about 1.3 GB, because its pressure-level chunks hold all 37 levels. Two starts a day read every 6-hourly state once. About 6.5 MB/s keeps one GPU busy for 168 h runs, about 15 MB/s for leads ≤ 72 h. WeatherBench2's 13-level copies end in January 2023 and lack skin and soil temperature, so they cannot start v1.
- **What we keep**: AIFS ENS's internal state and output fields, half precision, in object storage (Cloudflare R2 or Backblaze B2), never on the rented machine. One 1024-channel vector per station is about 0.6 MB per step per member, roughly 250 GB for 10 years. A regional patch over the stations, which a CNN-style head, new folds and pseudo-stations need, is about 3 MB and 1.25 TB. Output fields at the same nodes add about 10 % and serve the raw baseline and the output-field control.
- **Station truth**: temperature and dew point decoded from NOAA ISD (2015 to 2025-08-24) and OGIMET SYNOP bulletins after; the traps are in [estate.md](estate.md).

## Compute

Snapshot of 2026-09-30; prices move weekly.

- AIFS ENS v1 needs 38 GB of GPU memory (24 GB in chunked mode) and flash-attention (Ampere or newer). The AIFS-CRPS paper gives about 4 min for a 15-day member on one A100 40 GB, about 4 s per 6 h step.
- Cheapest suitable cards on Vast.ai: A100 80 GB from ~$0.87/h on demand, L40S 48 GB from ~$0.47/h; interruptible A100 80 GB from ~$0.17/h on verified hosts. Each start is a few minutes of work, so a pre-empted machine loses little.
- Rough sizes, 2 members, at 4 s per step:

| Run | Model steps | A100 hours | On demand, ~$0.87/h | Interruptible, ~$0.17/h |
|---|---|---|---|---|
| Pilot: 1–2 days of starts | < 1k | ~1 | ~$1 | < $1 |
| Training: 10 years, two starts a day, to 168 h | ~410k | ~455 | ~$395 | ~$77 |
| Training cut to leads ≤ 72 h | ~175k | ~195 | ~$170 | ~$33 |
| Scoring: 2025-07 → 2026-05 | ~37k | ~42 | ~$36 | ~$7 |

The pilot measures what these estimates guess: seconds per step, MB/s from Google Cloud to the host, MB per point, and peak GPU memory with the internal state captured.

## Reuse beyond AIFS ENS

The method is model-agnostic; what is trained is not. Any model whose weights we can run (AIFS Single, GraphCast, GenCast, Aurora, Aurora 1.5 ENS, WeatherNext 2) takes the backbone's place, with the head retrained. WN3 is closed; IFS and GFS are open source but have no learned internal state and cost too much to rerun. All three can still feed the head through their output fields around the point, closer to classic post-processing; whether that loses information is what the output-field control measures.

## Open design choices

- Which layer of AIFS ENS the head reads (the ~31 km decoder layer or the ~1° processor latent), and how many of its channels are kept.
- What to store: one vector per station, a stencil of nodes around each, or a regional patch.
- The training region: Leningrad Oblast alone, or 55.5–64°N, 20–42°E with the oblast also scored on its own.
- Start times: 00/12 UTC, or 06/18 UTC to cut ERA5's look-ahead from 9 h to 3 h.
- Which leads the head is trained on, and whether one head serves all leads.
- The GPU host and the object store.
