# GTA VI launch-day Twitch CCU forecast

Entry for [gta6-twitch-ccu-forecast-01](https://make.fun/c/gta6-twitch-ccu-forecast-01)
on make.fun — a program that forecasts, hour by hour, how many people watch
Grand Theft Auto VI live on Twitch over the 24 UTC hours of its release day
(2026-11-19), from everything recorded up to the end of its first hour.

`main.py` (Python 3.13, standard library only) reads every `<id>.in` in the
input directory and writes `<id>.out` to the output directory, exactly like
the kit's starter:

```
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Offline, deterministic, no model calls, no network. Runs in well under a
second per case (limit: 8,000 ms).

## Method

A rolling-midnight console launch has a distinctive audience shape: hour 1
only captures New Zealand / Australia, hours 2–4 dip as the West sleeps,
hours 5–6 jump when the US East unlocks, and the audience climbs to an
evening peak around hours 18–21 before decaying. The house baseline
forecasts hour 1's print for every hour, so it misses that whole shape.

Each hour's forecast is a log-normal centred on two anchors blended in log
space, both computed at run time from the peer launches shipped in the case
file (`comparables`):

1. **Hour-1 anchor** — `ln(p1) + median peer ln(v_h / v_1)`: the launch's
   own first print, carried along the peers' median ratio curve.
2. **Pre-launch-peak anchor** — `ln(prepeak) + median peer ln(v_h / prepeak)`,
   where `prepeak` is the highest hourly average in the 24 hours before the
   release day. A launch whose hour-1 print is small relative to its
   pre-launch hype still has its whole audience ahead of it; the pre-launch
   peak measures that hype directly and is far more stable across launches
   than the hour-1 print (peak/prepeak varies much less than peak/hour-1).

Blend: 30% anchor 1 / 70% anchor 2. Each median is itself a blend of two
peer sets: 75% console-only rolling-midnight peers (the launch's own kind —
PS5 / Xbox / Switch only, no PC) and 25% all rolling-midnight peers.

**Peer selection.** Rolling-midnight launches only (unlock text mentions
"midnight local" or "rolling"). Global-unlock launches are excluded: their
hour-1 print is near zero (unlock at mid-day UTC), which makes their ratios
meaningless for a rolling launch — including them lifts the median curve
and wrecks the forecast. A peer needs a complete 24-hour release day and a
usable pre-day peak; the GTA VI category's own pre-launch history entry is
excluded automatically (it has no release day).

**Spread.** `sigma_h = 0.85 ×` the cross-peer population standard deviation
of `ln(v_h / v_1)`, so the 90% interval is the peers' own disagreement
about that hour, and `q_tau = exp(mu_h + z_tau · sigma_h)` with
`z = (−1.645, −0.674, 0, 0.674, 1.645)`.

**Fallbacks.** If fewer than three rolling peers survive the filter, all
complete peers are used; if still fewer than three, or if anything goes
wrong reading a case, the program falls back to the house baseline formula
rather than emitting an invalid case.

## Practice validation

Four past console launches (Marvel's Wolverine, Red Dead Redemption 2, God
of War Ragnarök, Marvel's Spider-Man 2). Their release-day answers were
reconstructed from SullyGnome's hourly Twitch category averages and
cross-checked by reproducing the house baseline's published practice result
exactly (loss 0.387, typical miss 82.8%, coverage 0.783):

| model | pinball loss | typical miss | coverage_90 |
|---|---|---|---|
| house baseline | 0.387 | 82.8% | 0.783 |
| **this program** | **0.207** | **36.1%** | **0.902** |

Per case: Wolverine 0.258 (baseline 0.744), RDR2 0.132 (0.287), God of War
Ragnarök 0.217 (0.215), Spider-Man 2 0.220 (0.303).

Data: SullyGnome.com hourly Twitch category averages (factual public
counts, read once at low volume, with attribution in the case files).

## License

MIT — see `LICENSE`.
