# Speed curve validation — 2026-10-07

Validated on the fork's `codex/keyframe-animation` branch. This extends the
[intrinsic keyframe milestone](keyframes-validation.md). No upstream release,
PyPI publication, or replacement of the separate installed 0.16.0 MCP is implied.

## Automated results

- Python 3.12.13 / MCP 1.30.0: **2009 passed, 9 skipped**. Two existing
  `read_resource` return-type deprecation warnings.
- Python 3.13.15 / MCP 2.3.0: **2012 passed, 6 skipped**, no warnings.
- Whole-repository Ruff and `git diff --check` passed.
- Real subprocess stdio / ClientSession on both SDKs: initialization, tool
  discovery, native speed inspection, speed writing, intrinsic animation
  edits after retiming, reset, and readback. SDK 2 additionally validated all
  13 grouped schemas against Draft 7. The server exposes 95 handlers, 70 flat
  schemas, and 83 tools with legacy advertisement enabled.
- Tests cover exact NTSC time, holds and integrated ramps, nonzero asset and
  clip starts, invalid/boolean/nonfinite values, native rational limits,
  source bounds, ripple and ambiguous timing rejection, untouched input,
  atomic failures, bundle sidecars, no-overwrite publishing, DTD failure,
  journaling, and animation clock behavior.
- Independent review reproduced reset against a different source/local
  origin, native microsecond quantization, the exact reported snap changes,
  preserved animation positions and ripple offsets. The schema/core sampling
  mode mismatch discovered during review was corrected and rechecked.

The full-suite commands were `python -m pytest tests/ -q` in each environment.
Core tests are `tests/test_retime.py`; MCP workflow tests are
`tests/test_retime_tools.py`. Intrinsic interaction tests extend
`tests/test_keyframes.py`. No new runtime dependency was introduced.

## FCP 12.4 import, playback and export

All media is synthetic, using the existing 12-second 1280x720 / 30 fps test
pattern and 440 Hz audio. The separate validation library was used; original
user projects were not edited. Imports used warning suppression **off**.

The final project, `Speed Curve Final QA`, starts with a six-second clip from
source 5–11 seconds, followed by a two-second clip. A curve with control
points `(0s,1x), (2s,2x), (4s,1x)` shortens the first clip to four seconds.
It emits 121 linear native time-map points and consumes the same six seconds
of source. The following clip moves from output 6s to 4s; sequence duration
changes from 8s to 6s. FCP imported it without warnings. Its retime editor
showed the 120 short speed segments, rising toward 200% and falling toward
100%, consistent with the documented per-frame sampling.

After retiming, the five intrinsic curves were written through MCP at
relative 0s and 3s. At output 1.5s, FCP Inspector showed:

| Property | Endpoints | Observed midpoint |
|---|---|---|
| Position | `[0,0]` → `[10,5]` | 36px, 18px |
| Scale | 1 → 1.2 | 110% |
| Rotation | 0 → 15 degrees | 7.5 degrees |
| Opacity | 0.25 → 1 | 62.5% |
| Volume | -24 → 0 dB | -5.5 dB (native gain interpolation) |

Playback was observed running and then stopped at the final frame, 5:29.
The project was exported through FCP's own Export XML command as FCPXML1.14.
The export passed Apple's installed DTD. At all 121 project-frame boundaries,
the exported source-time map matched the intended map within
`277791/119800000000` seconds (approximately **2.32 microseconds**, far below
one 1/30-second frame). All ten intrinsic endpoint times and values were
unchanged. The following clip's offset remained 4s and sequence duration 6s.
The exported map could be reset to source 5–11s at 1x, with its tiny source
in/out adjustments explicitly reported by `reset_speed`.

An earlier constant-2x probe established a critical clock distinction:
`timeMap` remaps media time, while intrinsic keys stay on the adjusted local
clock. At output1.5s with clip.start5s and keys5–10s, rotation was4.5degrees
and volume-9.3dB, matching local time6.5s rather than mapped source time8s.
The implementation and sanitized regressions follow this observed behavior.

## Evidence and limits

Gitignored local evidence under `.local-validation/keyframes/`:

- `retime-mcp1-stdio-result.json`
- `retime-qa-baseline_retimed_keyframes.fcpxml`
- `retime-final-roundtrip.fcpxmld/Info.fcpxml`
- `retime-final-roundtrip.json` (hashes, exact errors, reset report)
- `retime-coordinate-roundtrip.fcpxmld/Info.fcpxml`

SDK2 stdio and full-suite logs were retained in `/private/tmp` on the validation
machine. Raw exports with bookmarks and local media paths are not in Git.

The pitch-preservation flag is written and native pitch-on behavior is the
FCP default; this run verifies configuration and playback, not subjective
pitch fidelity or artifact-free speech. Optical-flow quality, reverse,
freeze frames, sparse native smooth2 curves, compound/multicam structures,
and attached items on the retimed target are outside this implementation's
verified scope. Proxy renders do not validate these speed or animation effects.
