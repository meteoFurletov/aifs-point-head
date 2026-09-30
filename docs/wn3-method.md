# The WeatherNext 3 station head

What this repo copies, taken from the paper. Section and equation numbers refer to it.

Rasp, S., Babenko, B., Masters, D., El-Kadi, A., et al. (2026). *WeatherNext 3: Increasing resolution and performance of global weather models with raw observations.* Google DeepMind. arXiv [2609.03582](https://arxiv.org/abs/2609.03582); [PDF](https://storage.googleapis.com/deepmind-media/papers/weathernext_3.pdf). The PDF is © Google and is not kept in this repo.

## The problem it solves

AI weather models are trained on and started from analyses, so they inherit the analyses' biases, surface temperature among them (§1). Weather stations are the better truth for what people feel at the ground, but they are sparse and irregular, which gridded models cannot train on directly. WN3 adds an output that is trained on station reports yet can be queried anywhere.

## The split: big backbone, small head

- **Backbone.** A mesh-transformer ensemble model (the FGN family of WeatherNext 2), trained on dense analyses over decades: ERA5 for 1959–2015 and ECMWF HRES analyses from 2016, at 1°, then 0.25°, then 0.1° (0.1° from HRES only, 2016 on) (§2.3, Table A.1). This is where the representation of the atmosphere comes from.
- **Station head.** A small network that decodes the backbone's state at any point and time (eq. A.3, Appendix A.1.1):
  1. a 4-layer, width-768 CNN runs over the backbone's decoded 0.1° latent grid, together with the input frames (skip connection);
  2. the result is bilinearly interpolated to the query location;
  3. it is concatenated with a metadata embedding: an MLP (one hidden layer, width 768) over **station elevation, land or sea, and the query time within the 6 h step**;
  4. an MLP (two hidden layers, width 768) outputs **2 m temperature and 2 m dew point**.

Because the query is continuous, the head predicts at any location and time. The operational 0.05° hourly product is this head queried on a grid; querying at exact station coordinates only helped marginally (§2.2).

## How the head is trained

- **Frozen fine-tuning.** The head is trained with the backbone and all other heads fixed, at 0.1°. It runs in two stages interleaved with the backbone's autoregressive fine-tuning: 75 % of the head's steps after the 7-step rollout stage, then, after the final 8-step stage, 25 % more that merge the final backbone with the head's weights (Appendix A.1.3).
- **Data.** METAR and Mesonet (both via MADIS) plus ICOADS ships and buoys, from 2001: about 5,000 airports, about 15,000 Mesonet stations in 2024, about 3,000 marine reports an hour (§2.3).
- **Cleaning.** Mesonet keeps the report nearest the hour; METAR and Mesonet use their own quality flags. ICOADS has no flags, so its values outside −20 to 40 °C are dropped. For training only, any report more than 5 °C from ERA5 is dropped, in all three sources (Appendix A.3.3).
- **Metadata anywhere.** Elevation is a 0.05° grid-box mean of GMTED2010; land/sea is ESA WorldCover 10 m, averaged the same way (Appendix A.3.4).
- **Held-out stations.** A random but temporally fixed 5 % of METAR and Mesonet stations is never trained on and is used for evaluation only (§2.3).
- **Pseudo-stations.** Each hour, 2,000 random points are filled from ERA5 or HRES and added as extra targets. This removed strong biases in sparsely observed regions (Andes, Himalaya, high-latitude oceans) without hurting held-out skill (§2.3).
- **Loss.** Fair CRPS over two sampled trajectories. For each data type the loss is averaged over that type's own points, so sparse stations are not swamped by gridded fields; the station variables `2t` and `2d` have weight 1.0 (eq. A.4, Appendix A.1.1; Table A.3). The globally pooled CRPS used for precipitation and cloud was left off for stations, because it made mesh-scale artefacts worse (Appendix A.1.2).
- **Wind** station reports are used for evaluation only. **Precipitation** is learned from satellite and radar products (IMERG, PARDIG), not from gauges.

## Ensemble

The spread comes from the backbone: a low-dimensional noise vector perturbs its normalisation layers, plus "epistemic dropout" with a mask drawn per member and time step at inference (§2.2). The head adds no randomness; it decodes each member.

## Evaluation

- Against the held-out METAR and Mesonet stations, bilinearly interpolated from the 0.05° query grid; nearest grid point scored worse (§3.1).
- The lapse-rate height correction applied to the 0.25° and 0.1° fields made a negligible difference for the head, so its scores are raw (§3.1, §4.2).
- Relative humidity is computed from temperature and dew point (§3.1).
- Model versions are trained on data up to the start of each evaluation year, so scored years are never seen (Appendix A.1.3).

## Results

- On unseen stations at short leads, 2 m temperature CRPS improves by up to **30 % on WeatherNext 2 and 40 % on ECMWF ENS**; relative humidity improves similarly against ENS (§4.2).
- Skill at training stations is only a few percent better than at held-out ones: the head generalises in space (§4.2, Figure A.5).
- In the real-time comparison, 1 July to 11 August 2026, the head clearly beats **AIFS ENS** for 2 m temperature and relative humidity at stations (§4.5, Figure 6c).

## Known artefacts

- Jumps in temperature at each 6 h step boundary, and a per-member global bias: one member runs warm or cold everywhere, redrawn each 6 h step (§4.6, Figure 7c).
- Quantiles and medians are largely free of both, except over Antarctica, where few stations constrain the head. The paper recommends ensemble statistics over single members.
