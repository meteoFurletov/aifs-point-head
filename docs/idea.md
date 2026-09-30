# The idea: WN3's head on AIFS ENS

What this repo builds, and where it has to differ from WN3. The method itself is in [wn3-method.md](wn3-method.md); names are in [estate.md](estate.md).

## Why

AIFS ENS 2 m temperature is a grid-box value at ~31 km and misses what a thermometer at a given spot reads. Learning that correction from AIFS ENS output alone does not work: it has run operationally only since July 2025. WN3 shows the way round: let a model trained on decades of analyses carry the representation, keep it frozen, and fit only a small head to station reports. AIFS ENS is open (weights CC BY 4.0, code Apache 2.0) and was pre-trained on ERA5 1979–2022, so it can be that frozen model.

## WN3 against this repo

| | WN3 | aifs-point-head |
|---|---|---|
| Backbone | WN3's own mesh transformer, trained by DeepMind | AIFS ENS, used as is, never trained |
| What the head reads | the decoded 0.1° latent | AIFS ENS's last internal layer on its ~31 km grid |
| Metadata | elevation, land/sea, time in the step | the same, plus lat/lon if it helps |
| Outputs | 2 m temperature, 2 m dew point | the same |
| Stations | METAR, Mesonet, ICOADS, global, ~20k | SYNOP 2025–26 and ISD 2015–24, Leningrad Oblast first (~240 training stations) |
| Held out | a fixed 5 % of stations | spatial folds of stations, built here |
| Pseudo-stations | 2,000 an hour from ERA5/HRES | none at first; added only if held-out skill between stations is poor |
| Loss | fair CRPS, 2 members | the same |
| Ensemble | from the backbone | from AIFS ENS's members |

## What has to be different

- **We run AIFS ENS ourselves.** ECMWF publishes output fields, not internal state, so the head's inputs come from our own reruns, started from ERA5. Reruns do not match ECMWF's forecasts bit for bit, so the fair comparison is head against raw on the same reruns.
- **A coarser internal grid.** WN3's head reads a 0.1° grid; ours ~31 km. Local detail (lakes, coast, relief) has to come from the metadata; this is the biggest risk.
- **Few stations.** About 240 training stations against WN3's ~20k. Spatial generalisation is the thing to test, not assume.
- **One AIFS ENS version throughout.** v1 needs fewer inputs than v2 (no 10 hPa level, no ocean waves).

## Data

- **ERA5 starting states**, streamed and never stored: each start reads two global states (the 6 h before and now), about 1 GB, from ARCO-ERA5 or WeatherBench2's 13-level copy on Google Cloud. Starts at 00 and 12 UTC read every 6-hourly state once. About 5 MB/s keeps one GPU busy.
- **What we keep**: AIFS ENS's internal state at the station points only, half precision, about 0.6 MB per step per member; roughly 250 GB for 10 years. Kept in object storage (Cloudflare R2 or Backblaze B2), never on the rented machine.
- **Station truth**: temperature and dew point decoded from NOAA ISD (2015–24) and OGIMET SYNOP bulletins (2025–26); the traps are in [estate.md](estate.md).

## Compute

Snapshot of 2026-09-30; prices move weekly.

- AIFS ENS needs 38 GB of GPU memory (24 GB in chunked mode) and flash-attention (Ampere or newer). ECMWF quotes ~2.5 min for a 10-day AIFS forecast on one A100, about 3–4 s per 6 h step.
- Cheapest suitable cards: A100 80 GB from ~$0.67/h on Vast.ai; L40S 48 GB from ~$0.39/h.
- Rough sizes, 2 members:

| Run | Model steps | A100 hours | Cost at ~$0.70/h |
|---|---|---|---|
| Pilot: 1–2 days of starts | < 1k | ~1 per card | ~$2 |
| Training: 10 years, 00/12 UTC, to 168 h | ~400k | ~400 | ~$280 |
| Training cut to leads ≤ 72 h | ~175k | ~170 | ~$120 |
| Scoring: 2025-07 → 2026-05 | ~37k | ~36 | ~$25 |

The pilot measures what these estimates guess: seconds per step, MB/s from Google Cloud to the host, and MB per point.

## Reuse beyond AIFS ENS

The method is model-agnostic; what is trained is not. Any model whose weights we can run (AIFS Single, GraphCast, GenCast, Aurora) takes the backbone's place, with the head retrained. Closed models (WN3, IFS, GFS) can still feed the head through their output fields around the point, which loses information and is closer to classic post-processing.

## Open design choices

- Which layer of AIFS ENS the head reads, and how many of its channels are kept.
- Which leads the head is trained on, and whether one head serves all leads.
- The GPU host and the object store.
