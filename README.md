# ld-decode-testdata-ci

This is a "light" version of ld-decode-testdata with only the files needed to run CI/CD.

Please do not add anything to this not directly used for CI/CD.

This is intended for non-infringing/fair use research usage to support ld-decode development.

## The radius sweep

`radius/` holds eighteen cuts taken from six discs at three radii each, plus one extra. They are
needed by CI/CD: the modulation transfer function of a LaserDisc changes with radius, so a decoder
can pass every conformance check at one radius and fail at another, and a suite that only ever sees
one radius cannot tell the difference between a correct decoder and a decoder tuned to one part of
one disc. The VITS conformance suite (`analysis/vits_conformance.py`) runs against every cut.

The extra is `domesday-ds1-community-north-outer.ldf`, kept for one behaviour no other cut has. Its
multiburst finds the chroma band already flat, so the video EQ servo declines inside its dead-band
and never adopts — and the bound the multiburst places on the inverse-MTF correction used to be
published only at such an adoption, leaving the burst servo to wind unbounded to 1.418 and fail
twenty of forty-nine conformance checks. DD86-DS2, the pressing that replaced DS1 in the sweep,
does adopt at every radius and so cannot catch the regression. Only the outer cut is kept: DS1
inner and middle behave as their DS2 counterparts do, and each costs 30 MB in a repository CI
clones.

**Decision, taken 2026-09-02:** grow this repository rather than split the sweep into a second
repository, hold it as release assets, or leave it on a maintainer's machine. The sweep is
CI-critical rather than optional, so a lane that could silently skip it would defeat the purpose;
one repository keeps the captures and the manifest that describes them versioned together. The cost
is accepted knowingly: the repository goes from 196 MB to 492 MB, and every checkout, clone and
cache restore carries it. Closing the single-image gates later the same day added a further 146 MB
of captures on the same reasoning: a gate resting on one disc image is a gate that cannot fail for
the right reason. The two lanes that cost no capture data at all were taken first, and the six cuts
were chosen as the fewest that leave no gate on one image.

Each cut is 30 frames (PAL) or 20 frames (NTSC), sized so that a decode yields at least ten
same-parity fields, which is what the coherent averaging in the conformance runner needs. Cuts are
taken at 5 %, 50 % and 95 % of the *recorded band* — that is, past the spin-up offset between the
start of the capture and disc frame 1, which is measured rather than assumed.

## No gate on one disc image

The sweep began as four discs, and that left rows of the conformance baseline resting on a single
capture. Two allowances — `multiburst_flatness` and `multiburst_out_of_band_response` on NTSC —
were measured on `ggv1069-side1-inner` alone, because the FCC multiburst is the only NTSC train
whose amplitudes are admissible at this field count and no other capture here carried it. A row
measured on one cut cannot tell an allowance that holds from an allowance that happens to fit one
disc.

Two discs and two lanes close that, added 2026-09-02:

- **`ggv1069-side1-ldv4300d-*`** — the same GGV1069 pressing as `ggv1069-side1-*`, read on a
  different player six years later. Same pressing, different capture chain: where the two disagree
  the difference is the capture and not the decoder, which nothing else here can establish. Its
  inner cut is a second FCC multiburst.
- **`industrial-lv-side1-*`** — Pioneer's Industrial LaserDisc/LaserVision System disc, a
  non-Domesday PAL pressing. Every PAL row of the baseline was measured on GGV1011 and Domesday, so
  a fault of the Domesday mastering and a fault of the decoder read the same. It is also the first
  disc here carrying the `PAL_MULTIBURST_IEC` frequency set — packets 1 to 5 land within 0.07 MHz
  of the IEC nominals, where every other PAL disc here carries the ITU set.
- **`ntsc/ggv-ntsc-mb-v2800.ldf`** and **`ntsc/ve-monitor.ldf`** are judged by the conformance lane
  as whole captures. Both were already here with nothing reading them, so they cost no new capture
  data; `ggv-ntsc-mb-v2800` is a third FCC multiburst reading and `ve-monitor` decodes to 178
  fields, more than four times any radius cut.

Discs rejected while choosing these, recorded so the work is not repeated. National Gallery of Art
sides 1 and 2 carry every NTSC luminance level about 7 % low at all three radii — white reference
92.9 IRE against 100, the 50 IRE zone at 46.1, the grey pedestal at 45.3 — while every chrominance
level passes, so its absolute-level checks fail for a reason that is the pressing's and not the
decoder's. Dragon's Lair, Space Ace, Firefox and Apple Visual Almanac hold no fixed offset between
file frame and disc frame; on Dragon's Lair it drifts from 818 to 8071 file frames across the side,
so a position in the file no longer names a radius and a "95 %" cut would not be one.

## vits-manifest.json

`vits-manifest.json` records what VITS each capture in this repository actually carries, measured
by `analysis/vits_inventory.py` rather than assumed from the line numbers the standards state —
real discs move these signals, and three of the six discs here do.

The conformance runner reads it (`--manifest`) so that a check it never got to attempt is reported
as `skipped: capture carries no <signal>` instead of passing over in silence, and so that a capture
with no entry is refused outright: an unsurveyed capture and a survey that has gone stale look the
same from the runner's side, and neither may be judged as if it were known.

Regenerate it with:

    python3 analysis/vits_inventory.py <capture.ldf> --system PAL|NTSC \
        --probe-percentages 0 --json entry.json

A capture added without a manifest entry will fail CI.

## What every capture here carries

Rendered from `vits-manifest.json` by `analysis/vits_inventory.py --manifest-table`, so it cannot
drift from the data beside it. Regenerate with:

    python3 analysis/vits_inventory.py --json-in testdata/vits-manifest.json --manifest-table

"Disc frame" is where inside the disc a cut begins; a full-disc capture begins in the lead-in and
reports its spin-up instead. The VITS column gives the **field** line each signal was found on and
the parities that carried it — found by content, never assumed from the line number the standards
state, because real discs move these signals and several of these do. The provenance of the
captures that predate this survey is recorded nowhere, so it is left blank rather than guessed.

| File | Disc | Side | System | Format | Source library | File frames | Disc frame | Radius | Spin-up | VITS |
|---|---|---|---|---|---|---|---|---|---|---|
| `ntsc/ggv-ntsc-mb-v2800.ldf` | — | — | NTSC | CAV | — | — | 999 | — | from disc frame 999 | `ntsc-fcc-multiburst` line 22/23/24 F1F2; `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `ntsc/issue176.ldf` | — | — | NTSC | CLV | — | — | — | — | ? | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `ntsc/ve-monitor.ldf` | — | — | NTSC | CAV | — | — | 46616 | — | from disc frame 46616 | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `ntsc/ve-snw-cut.ldf` | — | — | NTSC | CAV | — | — | 30249 | — | from disc frame 30249 | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `pal/ggv-mb-1khz.ldf` | — | — | PAL | CAV | — | — | 754 | — | from disc frame 754 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 13 F1; `pal-multiburst-field2` line 13 F2 |
| `pal/jason-testpattern.ldf` | — | — | PAL | CAV | — | — | 52498 | — | from disc frame 52498 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 13 F1; `pal-multiburst-field2` line 13 F2 |
| `pal/kagemusha-leadout-cbar.ldf` | — | — | PAL | ? | — | — | — | — | ? | none identified |
| `radius/dolby-surround-side1-inner.ldf` | Checking Dolby Surround | 1 | NTSC | CAV | LDV4300D_1/NTSC/CheckingDolbySurround | 2804–2823 | 2350 | inner (5 %) | 454 | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/dolby-surround-side1-middle.ldf` | Checking Dolby Surround | 1 | NTSC | CAV | LDV4300D_1/NTSC/CheckingDolbySurround | 23959–23978 | 23505 | middle (50 %) | 454 | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/dolby-surround-side1-outer.ldf` | Checking Dolby Surround | 1 | NTSC | CAV | LDV4300D_1/NTSC/CheckingDolbySurround | 45114–45133 | 44660 | outer (95 %) | 454 | `ntsc-ntc7-combination` line 20 F2; `ntsc-ntc7-composite` line 20 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/domesday-ds1-community-north-outer.ldf` | BBC Domesday DD86-DS1 Community North | 1 | PAL | CAV | BBC_AIV/Domesday/Domesday_DS1 | 51657–51686 | 51312 | outer (95 %) | 345 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/domesday-ds2-community-north-inner.ldf` | BBC Domesday DD86-DS2 Community North | 1 | PAL | CAV | BBC_AIV/Domesday/Domesday_DS2 | 3036–3065 | 2700 | inner (5 %) | 336 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/domesday-ds2-community-north-middle.ldf` | BBC Domesday DD86-DS2 Community North | 1 | PAL | CAV | BBC_AIV/Domesday/Domesday_DS2 | 27336–27365 | 27000 | middle (50 %) | 336 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/domesday-ds2-community-north-outer.ldf` | BBC Domesday DD86-DS2 Community North | 1 | PAL | CAV | BBC_AIV/Domesday/Domesday_DS2 | 51636–51665 | 51300 | outer (95 %) | 336 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/ggv1011-side1-inner.ldf` | Pioneer GGV1011 | 1 | PAL | CAV | Calibration/GGV1011 | 1429–1458 | 1200 | inner (5 %) | 229 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 13 F1; `pal-multiburst-field2` line 13 F2 |
| `radius/ggv1011-side1-middle.ldf` | Pioneer GGV1011 | 1 | PAL | CAV | Calibration/GGV1011 | 12236–12265 | 12007 | middle (50 %) | 229 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 13 F1; `pal-multiburst-field2` line 13 F2 |
| `radius/ggv1011-side1-outer.ldf` | Pioneer GGV1011 | 1 | PAL | CAV | Calibration/GGV1011 | 23042–23071 | 22813 | outer (95 %) | 229 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 13 F1; `pal-multiburst-field2` line 13 F2 |
| `radius/ggv1069-side1-inner.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | Calibration/GGV1069 | 1545–1564 | 1260 | inner (5 %) | 285 | `ntsc-fcc-multiburst` line 22/23/24 F1F2; `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/ggv1069-side1-ldv4300d-inner.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | LDV4300D_1/NTSC/Calibration Discs | 1560–1579 | 1265 | inner (5 %) | 295 | `ntsc-fcc-multiburst` line 22/23/24 F1F2; `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/ggv1069-side1-ldv4300d-middle.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | LDV4300D_1/NTSC/Calibration Discs | 12945–12964 | 12650 | middle (50 %) | 295 | `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/ggv1069-side1-ldv4300d-outer.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | LDV4300D_1/NTSC/Calibration Discs | 24330–24349 | 24035 | outer (95 %) | 295 | `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/ggv1069-side1-middle.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | Calibration/GGV1069 | 12888–12907 | 12603 | middle (50 %) | 285 | `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/ggv1069-side1-outer.ldf` | Pioneer GGV1069 | 1 | NTSC | CAV | Calibration/GGV1069 | 24231–24250 | 23946 | outer (95 %) | 285 | `ntsc-ntc7-combination` line 13 F2; `ntsc-ntc7-composite` line 13 F1; `ntsc-virs-field1` line 19 F1; `ntsc-virs-field2` line 19 F2 |
| `radius/industrial-lv-side1-inner.ldf` | Pioneer Industrial LaserDisc LaserVision System | 1 | PAL | CAV | LDV4300D_1/PAL/The Industrial LaserDisc LaserVision System | 2235–2264 | 1930 | inner (5 %) | 305 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/industrial-lv-side1-middle.ldf` | Pioneer Industrial LaserDisc LaserVision System | 1 | PAL | CAV | LDV4300D_1/PAL/The Industrial LaserDisc LaserVision System | 19614–19643 | 19309 | middle (50 %) | 305 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
| `radius/industrial-lv-side1-outer.ldf` | Pioneer Industrial LaserDisc LaserVision System | 1 | PAL | CAV | LDV4300D_1/PAL/The Industrial LaserDisc LaserVision System | 36992–37021 | 36687 | outer (95 %) | 305 | `pal-blanked-field1` line 22 F1; `pal-blanked-field2` line 22 F2; `pal-its-field1` line 19 F1; `pal-its-field2` line 19 F2; `pal-multiburst-field1` line 20 F1; `pal-multiburst-field2` line 20 F2 |
