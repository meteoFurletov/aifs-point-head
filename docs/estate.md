# Estate

Owner: Nikita Furletov. The one place for the names and facts every stage reads before
drafting. A correction that lands in chat twice lands here once.

## Names

- **AIFS ENS**: ECMWF's AI ensemble forecast, ~31 km, run by us from its open weights
  (HF `ecmwf/aifs-ens-1.0`, `ecmwf/aifs-ens-2.0`; CC BY 4.0) with Anemoi. The backbone,
  kept frozen. Not "AIFS" alone, which also names the deterministic AIFS Single.
- **Head** (station head): the small network this repo trains. It reads a frozen model's
  internal state at a point plus local geography and outputs 2 m temperature and dew
  point there, fitted to station reports. Retrained per model; the method is shared.
- **Backbone**: the frozen forecast model under the head. AIFS ENS first; any model whose
  weights we can run can take its place.
- **WN3**: Google DeepMind's WeatherNext 3, whose station head this repo copies (Rasp et
  al. 2026, arXiv 2609.03582). A method source; its past forecasts can serve as a
  benchmark for the head.
- **ERA5**: ECMWF global reanalysis; the starting states for our AIFS ENS runs, read from
  ARCO-ERA5 on Google Cloud.
- **SYNOP / ISD**: station truth. SYNOP is raw WMO FM-12 bulletins from OGIMET (2025–26);
  ISD is NOAA's archive of the same reports (2015 to 2025-08-24).
- **OSCAR**: WMO's station registry; the source of station coordinates and elevation.
- **dynamical.org**: public Zarr (Icechunk) archive of ECMWF's operational AIFS ENS (from
  2025-07-02) and IFS ENS (from 2024-04-01) forecasts: 51 members, 0.25°, 6-hourly leads,
  output fields only.

## Facts

- AIFS ENS v1 needs a GPU with 38 GB (24 GB in chunked mode); ECMWF gives no figure for
  v2. Both need flash-attention (Ampere or newer). Cloud agent sessions have no GPU; runs
  go to a rented GPU.
- ECMWF publishes AIFS ENS output fields, not its internal state, so we rerun it
  ourselves; reruns do not match ECMWF's forecasts bit for bit.
- Only AIFS ENS v1 can start from ERA5: v2 also needs six wave-period heights
  (h1012–h2530) that ERA5 does not have. v1 ran operationally from 2025-07-01 to
  2026-05-12, v2 since.
- AIFS ENS v1 was pre-trained on ERA5 and fine-tuned on operational analyses 2016–2023,
  so its runs in those years are in-sample for the backbone. ECMWF's sources disagree on
  the ERA5 years: 1979–2017 (v1 page and paper) or 1979–2022 (v2 card).
- AIFS ENS computes on a ~1° grid (O96); only its decoder's last layer is on the ~31 km
  N320 grid.

- Leningrad Oblast with St Petersburg has 13 SYNOP stations with ISD data in every year
  2015–24 (17 in OSCAR). About 250 stations need roughly 55.5–64°N, 20–42°E, nearly half
  of them Finnish and reporting hourly.
- OGIMET's getsynop filters by any WMO-index prefix, state or lat/lon box, and returns at
  most 200,000 reports per request. Space queries 20 s apart: a throttle users report,
  not a documented rule.
- ISD stopped at 2025-08-24 and is superseded by GHCNh; read it from NOAA's S3 bucket
  `noaa-global-hourly-pds`. SYNOP from OGIMET covers everything after.
- ISD and OGIMET SYNOP are the same reports decoded the same way (checked July 2020: bias
  +0.001 °C), so the two periods join without a seam.
- ISD merges some SYNOP stations with a nearby airport's METAR under one ID: 26063 holds
  the city SYNOP (Voeikovo, FM-12) and Pulkovo airport's METAR (FM-15), 18.6 km apart and
  about 1 °C apart at night. Keep FM-12 only, and take coordinates and elevation per WMO
  index from OSCAR, never from the ISD header.
- Station IDs stay strings: WMO IDs can start with 0.
- Stations are held out by location, because the claim is skill where no station is;
  scoring is also out of time, after the backbone's training data. AIFS ENS never trained
  on station reports, and ERA5 assimilates held-out stations like any other. ERA5's
  00 and 12 UTC states use observations up to 9 h later; its 06 and 18 UTC states up to
  3 h, as operational analyses do.
- ERA5 is under the Copernicus licence (attribute C3S). Station reports from outside the
  US fall under WMO Resolution 40: keep station files out of git and public buckets, and
  publish derived scores only.

- WN3 licence: download only runs whose whole 15-day window ended more than 1 h ago.
  Everything held is then CC BY 4.0: credit "WeatherNext 3, Google DeepMind", link the
  licence and say what was changed. Newer data falls under the GDM Real-Time Experimental
  Data Terms of Use (sharing limits, a set citation text, revocable); keep none of it.

## How changes land

- One branch per change, a pull request to `main`; Nikita merges.
- Roll back by reverting the pull request.
