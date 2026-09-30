# aifs-point-head

Turn a weather model's forecast into forecasts at any point, fitted to what weather
stations report. A small network, the **head**, reads the model's internal state at a
location plus that location's geography and outputs 2 m temperature and dew point there.
The model underneath stays frozen. This is the station-head method of WeatherNext 3
(Rasp et al. 2026, arXiv 2609.03582), made reusable across models.

First model: ECMWF's open **AIFS ENS**. First region: Leningrad Oblast.

Background: [docs/wn3-method.md](docs/wn3-method.md) (what WN3 does) and [docs/idea.md](docs/idea.md) (what this repo builds).

Work runs through the balka loop; see `AGENTS.md` and `docs/estate.md`.
