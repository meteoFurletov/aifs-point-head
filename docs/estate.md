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
  al. 2026, arXiv 2609.03582). A method source, not a data source.
- **ERA5**: ECMWF global reanalysis; the starting states for our AIFS ENS runs, read from
  ARCO-ERA5 on Google Cloud.
- **SYNOP / ISD**: station truth. SYNOP is raw WMO FM-12 bulletins from OGIMET (2025–26);
  ISD is NOAA's archive of the same reports (2015–24).
- **Sibling**: GitHub `meteoFurletov/aifs-cerra-downscaling`, which holds the Leningrad
  Oblast stations, their spatial folds and the SYNOP/ISD decoding and traps. Link to it;
  do not copy it.

## Facts

- AIFS ENS needs a GPU with 38 GB (24 GB in chunked mode) and flash-attention (Ampere or
  newer). Cloud agent sessions have no GPU; runs go to a rented GPU.
- ECMWF publishes AIFS ENS output fields, not its internal state, so we rerun it
  ourselves; reruns do not match ECMWF's forecasts bit for bit.

## How changes land

- One branch per change, a pull request to `main`; Nikita merges.
- Roll back by reverting the pull request.
