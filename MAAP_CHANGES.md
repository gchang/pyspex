# Changes from upstream

Upstream: https://github.com/rmvanhees/pyspex, release tag `v1.4.19`, commit 9143f79d39bb0a5eeb02a57b43e378a8dff94125 (2025-10-27), by Richard van Hees (SRON), BSD 3-Clause License (Copyright (c) 2019-2025 SRON).

This copy starts from upstream's file tree at 9143f79 without its git history. Every tracked file is included (upstream has no `.github` folder), and the LICENSE and CITATION.cff are untouched. It is unchanged except for the items below. The operational L0 to L1A script `scripts/l1agen_spex.py` is not changed.

## 2026-10-24: h5yaml limited to versions older than 0.3

`pyproject.toml` asked for `h5yaml>=0.1.0` with no upper bound, so a fresh `pip install` now picks h5yaml 0.5, and `spx1_level01a` no longer works:

- h5yaml 0.4 and newer no longer has `h5yaml.conf_from_yaml`, so `src/pyspex/gen_l1a/l1a.py` fails at import (`ModuleNotFoundError`).
- h5yaml 0.3 changed the YAML layout that the L1A templates in `src/pyspex/Data/` are written in; `spx1_level01a` stops with `dictionary update sequence element #0 has length 16; 2 is required` and writes no product.

The dependency is now `h5yaml>=0.1.0,<0.3`. Upstream's development branch has moved the templates and the code to the new h5yaml API (from commit a735ade on), but that touches many files and is not released, so it is not taken here. With h5yaml 0.2.1, which pip now installs, the L1A product is identical in every dataset and attribute (apart from the creation date) to the one made with h5yaml 0.1.1, the version released together with pyspex 1.4.19. Results do not change.

`pyxarr` is still listed without an upper bound; pyspex 1.4.19 does not import it, and pip settles on pyxarr 0.4.2, the newest release that does not require h5yaml 0.4 or newer.

## 2026-10-24: heater P and I values stored as floats

`src/pyspex/Data/h5_nomhk_tm.yaml` declared `HTR1_CALCPVAL` to `HTR4_CALCPVAL` and `HTR1_CALCIVAL` to `HTR4_CALCIVAL` as `u4`, but the instrument sends them as 32-bit floats (`src/pyspex/lib/tmtc_def.py`, `>f4`). `spx1_level01a` therefore wrote them to `engineering_data/NomHK_telemetry` as unsigned integers: fractions were cut off and negative values, which are normal for the heater controller, came out as 0. The eight fields are now `f4`.

This changes results, on purpose: the heater P and I values in L1A products made with `spx1_level01a` are now the values in the telemetry (for example `HTR2_CALCPVAL` -4.109 and `HTR2_CALCIVAL` -0.01333 in the first packet of the test below) instead of 0. Nothing else in the product changes. Backported from the data type part of upstream commit 93295da564ed3ed7c562c4c5c04e71e7d48af7d9 (2026-09-22, "fixed floats in nomhk"); the dataset renames in that commit are left out. `scripts/l1agen_spex.py` has its own definition and was already correct.

## 2026-10-24: README

How to install from source: the version comes from the git tags (versioningit), so `pip install .` needs a clone that includes the tags.

## Not changed

- `spx1_level01a` still writes a slightly different layout from the operational product made by `scripts/l1agen_spex.py` (`science_data/science_hk`, `image_attributes/image_id` and `ICU_4P0VB_V` instead of `detector_telemetry`, `image_ID` and `ICU_4P0V_V`, and no `MPS_ACT_STATUS` or `SW_MAIN_LOOP_COUNT` in `NomHK_telemetry`). Upstream's development branch changes these; they are names and missing fields, not wrong values.
- `scripts/l1agen_spex.py --version` still says 1.4.18.
- `src/pyspex/gen_l1a/l1a.py` imports `typing.Self`, which needs Python 3.11, although `pyproject.toml` allows 3.10.

## Check

Clean install in a new conda environment (conda-forge: python 3.12, pip, h5py, netcdf4, numpy, xarray, pyyaml), from a local clone tagged `v1.4.19.post1`:

```sh
pip install .        # pyspex 1.4.19.post1, h5yaml 0.2.1, pyxarr 0.4.2
```

Input: the L0 file from the example README, `SPX000007333.spx` (535 MB, 2024-04-07 06:01 to 06:08 UTC), and the PACE HKT files `PACE.20240407T055523.HKT.nc`, `PACE.20240407T060101.HKT.nc` and `PACE.20240407T060639.HKT.nc` (39 MB each), all from OB.DAAC (`https://obdaac-tea.earthdatacloud.nasa.gov/ob-cumulus-prod-public/<file>`, Earthdata login). A YAML file with `processing_version: 3`, `compression: False`, `eclipse: False`, the L0 file and the three HKT files.

```sh
spx1_level01a --verbose info SPX000007333.spx                 # exit 0, about 1.3 GB peak memory
OCVARROOT=<dir with common/tai-utc.dat> python scripts/l1agen_spex.py --yaml run.yaml   # exit 0, 1.45 GB
spx1_level01a --yaml run.yaml                                  # exit 0, 1.28 GB
```

Each run takes seconds. Both L1A products were compared with the official granule `PACE_SPEXONE.20240407T054504.L1A.nc` (PACE_SPEXONE_L1A_SCI v3), matching records on time: all 1310 detector images are identical (0 of 266.6 million samples differ), as are the image attributes, the detector telemetry, the housekeeping telemetry (including the heater fields, for both programs), the temperatures and the navigation data, except that `att_quat` from `l1agen_spex.py` differs by 1 float32 unit in the last place in 581 of 1828 values, as before. The `l1agen_spex.py` result is the same as with the unmodified 1.4.19 release.
