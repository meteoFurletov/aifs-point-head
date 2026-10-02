# Intent: the bar set by AIFS ENS output fields

Author: Claude, for Nikita Furletov. Owner: Nikita Furletov. Status: draft. Date: 2026-10-01.

## Problem

Intent 001 asks whether a head on frozen AIFS ENS beats raw AIFS ENS at stations it never
saw. A model that knows each station's geography and learns a regional correction can beat
raw whatever the internal state holds, so a win over raw would not show that the internal
state helps. WN3 never measured this either. What a correction fitted on AIFS ENS output
fields alone achieves at unseen stations is the bar the head has to clear. ECMWF's
operational AIFS ENS has been archived since July 2025, so that bar can be measured before
any GPU is rented.

## Outcome

We know how much a station-trained correction of operational AIFS ENS output fields
improves 2 m temperature and dew point over raw AIFS ENS at stations it never saw, as an
ensemble, per lead, for Leningrad Oblast and its surroundings. We know it before paying
for an AIFS ENS rerun.

## Behaviours

- Operational AIFS ENS v1 output fields, all 51 members, are read at the station points
  for the period v1 was operational, 2025-07-02 to 2026-05-11.
- Station truth comes from ISD up to 2025-08-24 and OGIMET SYNOP after it, one report type
  per station, with coordinates from OSCAR.
- Raw AIFS ENS is interpolated to each station and corrected for height, as WN3 treats its
  baselines.
- Corrections are fitted at training stations from output fields and station geography,
  with no station identity: a regional EMOS and a small network.
- Everything is scored at stations held out by location and in time, with fair CRPS against
  raw, by a decision rule fixed before scoring.

## Out of scope

- Running AIFS ENS or reading its internal state (intent 001).
- Blending with IFS ENS.
- Other parameters than 2 m temperature and dew point.

## Open questions

- Training region: Leningrad Oblast alone (13 stations), or roughly 55.5–64°N, 20–42°E
  (about 250 stations, nearly half Finnish) with the oblast also scored on its own.
- Whether WN3's own 0.05° 2 m temperature and dew point, if Google grants access, is scored
  at the same stations as a second reference (it covers 2026 only).
