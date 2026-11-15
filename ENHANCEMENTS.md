# R1 Drift Racer — v8 Polish Pass (2026-11-15)

Surgical enhancements to the shipped v7 game. No engine rewrite, no save-key changes,
no control-scheme changes. All edits are in `index.html` (92,984 → 94,778 bytes).
Menu version label bumped to **V8**. Not pushed to GitHub — local only, ready for Jeff's call.

## Bugs found & fixed

1. **Restarting a Daily Challenge restarted the wrong race.** Pause → Restart, the
   desktop `R` key, and "Race Again" all called `setupRace()` with no arguments, which
   falls back to the *menu* track/direction/weather — so a daily's track, Reverse/Rain/
   Rush/Night modifiers were silently dropped on any restart. Fixed with an
   `activeRace` record: `setupRace` now stores the resolved config (def, reversed,
   weather, mod) of the race actually started, and Restart / `R` / Race Again replay
   that config. Cup races already worked (they resolve via the cup object) and still do.
   *Verified headlessly: today's daily (Square, Reversed) → pause → Restart → still
   Square, still reversed.*

2. **Boost pads fired while reversing, launching you forwards.** Pads triggered purely
   on proximity, so backing over one in R gear (or rolling backwards after a spin)
   slammed the car forwards at boost speed. Pads now require forward motion
   (>10 u/s along the car's heading). Forward pad behaviour is unchanged — bot lap
   times in the parity probe stayed in the same band after the fix.

3. **Engine audio droned forever on the results screen.** `update()` stops running
   when the phase leaves `race`, so `engine()` was never called again to mute the
   gain — the last engine note hung under the results panel until you quit to menu.
   `finishRace()` now explicitly silences the engine. Related: the engine note also
   ignored the gas latch (update passed a hard-coded `true`), so lifting off the gas
   never dropped the engine note. It now receives the real throttle state.

4. **Reversed Time Trial raced the wrong ghost.** Ghosts were keyed only by
   track+laps, so a normal-direction ghost played back (driving the wrong way round)
   in reversed TT, and a reversed run could overwrite your normal ghost. Ghosts are
   now keyed with a `:rev` suffix when reversed, for both saving and playback; the
   legacy un-suffixed key is still honoured for normal-direction playback, so
   existing saved ghosts keep working.
   *Verified headlessly: reversed TT saves `Oval:3:rev`, normal TT saves `Oval:3`.*

5. **Leaderboard panel survived a resume.** Open Leaderboard from pause, then resume
   via the side button or Esc, and the leaderboard stayed painted over the race.
   `resumeGame()` now hides the leaderboard along with the pause menu.

6. **Menu inputs leaked through the Garage.** While the garage overlay was open, the
   scroll wheel still cycled the menu track behind it (with beeps), and the side
   button / Enter started a race mid-browsing. Scroll, side-button and Enter are now
   ignored while the garage is open.

7. **Daily button contradicted itself.** If you'd already claimed today's daily,
   it toasted "Daily already completed — new one tomorrow!" and then started the
   daily anyway. It still lets you replay (for fun), but the toast now says so:
   "Daily already done ✓ — replay for fun (no reward)". The one-per-day reward
   guard in `finishRace` is unchanged.

## Performance fix

8. **Track scenery was re-rendered from scratch on every race start.** Each
   `setupRace` re-ran the full scenery painter (hundreds of canvas ops plus
   nearest-point scans over a ~700×700 offscreen canvas) even for a track you'd
   just raced. The scenery layer is now cached per track+direction+theme and
   reused; only the skid-mark canvas is reset (fresh canvas, same transform).
   The theme is part of the cache check because the Daily "Night" modifier
   re-themes a track — a night daily can never inherit the day scenery or
   vice-versa. This is the single biggest start/restart latency win available
   on R1 hardware, and restarts are now effectively instant.

## Gameplay / QoL improvements

9. **You can always steer out of a crash-stop.** Steering authority scaled from
   zero with speed, so after stopping nose-first against the scenery line you
   could barely rotate the car to escape — you had to reverse blind. Authority
   now has a 0.3 floor. At racing speeds the curve is untouched (still calms
   down at top speed as in v7), and all harness steering checks pass unchanged.

10. **Position tower always shows you.** The HUD tower listed only the top 4;
    running P5/P6 you vanished from your own standings. If you're outside the
    top 4, your row (with your real position number) replaces the P4 row.

11. **Controls reminder on the countdown screen.** A one-line hint under
    "tap to skip" — "🎡 wheel steers • side btn = gas • hold E-BRK to drift"
    (or the drag variant when a drag scheme is selected). New players no longer
    have to discover the R1 control scheme mid-race.

## Measured, and deliberately left alone

- **Bot lap-time parity.** Probed with an autopilot-driven player before and
  after (Oval + Volcano × Rookie/Pro/Master): bots and player land in the same
  lap-time band everywhere (e.g. Oval Pro bot bests 3.5–5.1s vs player 4.7s;
  Volcano Pro bots 7.8–10.0s vs player 10.9s with a deliberately crude
  autopilot). Occasional outlier bot laps trace to the intended rubber-banding
  plus boost-pad/draft chains. No difficulty retune — v7's balance held up.
- **Best times / best laps stay shared between Normal and Reverse.** Unlike
  ghosts (path data — plainly wrong cross-direction), a shared per-track
  record is a defensible design, the leaderboard is built around one row per
  track, and re-keying would orphan existing players' records. Left as-is.
- **Bots showing "DNF" on the results screen** when you finish first: the race
  ends at your finish by design in every mode; projecting bot finish times
  would add risk to the results flow for cosmetic gain. Left as-is.
- **Keyboard "Down = brake" in R gear** applies forward force (acts as a
  brake-to-stop). Desktop-only edge, behaves like a brake pedal; left as-is.
- **Economy**: chest cost 150 vs win payout 120 + a chest, pity timers at
  8/25 — verified working by the harness; no change needed.
- **282px menu fit**: CSS untouched except the version text in the existing
  subtitle line (same font/spacing, fits the 240px width); the v7 menu-fit
  fix cannot regress.

## Test results

- Existing `test-harness.js` (53 checks: systems soak on Volcano/Reverse/Rain,
  chests + pity, all customization combos, full Bronze Cup, daily, wheel/gear/
  E-BRK controls, pause settings, steering smoothing, drift tiers, drafting,
  collisions): **53/53 PASS before the pass, 53/53 PASS after** ("ALL V7 TESTS
  PASSED", run repeatedly — deterministic).
- One mid-pass failure ("reverse gear drives car backwards") led directly to
  bug #2: the new steering floor changed the test car's coast trajectory so it
  stopped on a boost pad and got launched forwards while reversing. Root cause
  was the pad, not the steering; fixed the pad.
- Extra regression probes (not in the shipped harness): daily-restart config
  retention, reversed/normal ghost key separation — both pass (see above).
- Still unverified on physical R1 hardware (unchanged from v7): scroll-wheel
  steering direction feel and the latching side-button accelerator — Jeff's
  device feedback is the only real test for those.
