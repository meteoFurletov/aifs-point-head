# Intent: a head on frozen AIFS ENS, after WN3

Author: Nikita Furletov. Owner: Nikita Furletov. Status: parked. Date: 2026-09-30.
Parked 2026-10-02: after the 31 October write-up in aifs-cerra-downscaling.

## Problem

Raw AIFS ENS 2 m temperature misses what thermometers read, and AIFS ENS has run
operationally only since July 2025, far too short a record to learn a correction from. WN3 showed a way round
that: keep a model trained on decades of analyses frozen, and fit only a small head on
top of it to station reports; the head then predicts at any point, even where no station
is. AIFS ENS is open and was trained on 40+ years of ERA5, so it can play that frozen model.

## Outcome

We know whether a head on frozen AIFS ENS beats raw AIFS ENS at stations it
never saw, for 2 m temperature and dew point, as an ensemble, per lead and season,
starting with Leningrad Oblast.

## Behaviours

- AIFS ENS runs from ERA5 on a rented GPU, and its internal state is kept only at the station points.
- The head learns 2 m temperature and dew point from SYNOP and ISD at the training stations.
- It predicts as an ensemble at any point, from AIFS ENS and local geography.
- It is scored at stations held out by location, per lead and season, against raw AIFS ENS.
- A small pilot measures cost and speed before any long run.

## Out of scope

- Downscaling to a regional reanalysis grid.
- Other parameters than 2 m temperature and dew point.
- Retraining AIFS ENS itself.

## Open questions

- None outstanding for the intent. Which GPU host, which AIFS ENS version and where the stored state lives are design choices.
