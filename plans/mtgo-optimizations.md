# docker-mtgo: performance, focus, and hygiene optimizations

## Goal
Make MTGO-in-docker faster and less annoying (windows stealing focus), and bring the
repo's scripts, Dockerfile, and CI up to current best practice. Every change is
packaged as an independent, PR-sized unit with a measurable before/after so it can
be argued upstream to `pauleve/docker-mtgo`; macOS and podman paths keep working,
and new behaviour is opt-in unless the default change is clearly safe.

## Context

Repo is tiny: `Dockerfile` (image), `extra/mtgo.sh` (in-container launcher,
installed as `/usr/local/bin/mtgo`), `run-mtgo` (host launcher), `sound/` (derived
image), `.github/workflows/`. Findings from scoping that drive the plan:

- **All Direct3D is software.** `Dockerfile:24` runs `winetricks renderer=gdi`, which
  writes `HKCU\Software\Wine\Direct3D\renderer=gdi` (verified in winetricks source,
  `load_renderer()` → `winetricks_set_wined3d_var`). It was added in 1f7c4c7 only to
  silence a libGL warning because the container has no GPU. MTGO is a WPF app; WPF
  renders through Direct3D 9, so every frame is CPU-rasterized. No `/dev/dri` or
  `--gpus` is ever passed by `run-mtgo`. Upstream issues #132 (NVIDIA) and #149
  (right-click dialogs peg CPU, "GPU not utilized") ask for exactly this.
- **Wine is two years old.** `FROM panard/wine:9.14-wow64` (Aug 2024). The base image
  `panard/wine` is at `11.2-wow64` (Feb 2026); upstream PR #204 / branch
  `origin/auto/wine-updates` (commit 71aa347) already contains the one-line bump.
  Local `panard/mtgo:latest` is even older (built 2024-05-04).
- **Known WPF CPU pinning fix exists.** The `phever/mtgo-linux` project (GE-Proton 11)
  documents that setting `DisableAutomationPeer`, `PurgeAutomationEvents`,
  `DisableStylusInput`, `DisableTabletDevices` to `true` in `MTGO.exe.config`
  (ClickOnce app dir under `.../Apps/2.0/...`) stops the "UI-Automation peer storm +
  stylus thread that pinned the UI thread". MTGO updates revert it, so it is re-applied
  before every launch. This matches the freeze symptoms in upstream #68/#107/#121/#149.
- **Focus stealing has a canonical Wine fix.** `HKCU\Software\Wine\X11 Driver\UseTakeFocus=N`
  (Wine bug 57585 proposes making it the Linux default). Nothing in this repo sets it.
  Upstream #196 (floating sub-windows moving/focusing on their own) was resolved by the
  maintainer with Wine's virtual desktop ("emulate desktop"), which today requires the
  manual `--winecfg` route (`README.md:73`). Winetricks has `vd=WxH` / `vd=off` verbs.
- **CPU pinning is fixed at `--cpuset-cpus 0-3`** (`run-mtgo:295`, default
  `limit_cpus=4`, added for #160). It errors on hosts with fewer than 4 CPUs and never
  scales up. On the dev host (Ryzen 7700X, 16 threads, siblings `0,8 1,9 …`) CPUs 0-3
  happen to be 4 distinct physical cores, so the pin is "correct" but small.
- **Per-launch overhead in `extra/mtgo.sh`:** `winetricks gdiplus=builtin sound=alsa
  winegstreamer=disabled wmp=disabled` runs on every start (`mtgo.sh:34-36`) even
  though the settings persist in the volume-backed `user.reg`; the watchdog loop
  (`mtgo.sh:69-81`) spawns `winedbg --command "info proc"` (a full Wine process) every
  6 s; `workaround_dotnet` (`mtgo.sh:52-63`) curls `mscoreei.dll` from GitHub at
  runtime although `extra/mscoreei.dll` is in the repo and could be baked in.
- **`run-mtgo` bugs:** leftover `set -x` in the macOS branch (`run-mtgo:61`);
  unquoted `[ -n ${_tz} ]` (`run-mtgo:169`, always true); `docker info` dumps its
  full output every run (`run-mtgo:175`); no `pipefail`, so `docker run` exit status
  is lost behind `| watch_openurl` (`run-mtgo:335`); no INT/TERM trap, which is why
  README has a "never exits after Ctrl+C → `docker kill mtgo_running`" entry; a stale
  `mtgo_running` container makes the next run fail with a name clash.
- **Podman:** upstream PR #215 shows the needed bits (`--userns keep-id`, pass `${opts}`
  to the volume-init `docker run` calls, `limit_cpus=0`) but hard-switches the default
  client to podman, which is why it is unmerged. README FAQ tells podman users to edit
  the script by hand.
- **Dockerfile/CI hygiene:** legacy `ENV K V` syntax (BuildKit warns), duplicate
  `USER wine`, no `.dockerignore`; workflows use `actions/checkout@v1`/`@v2`,
  `actions-hub/docker@master`, `peter-evans/create-pull-request@v3.5.0` (Node 12/16
  actions, deprecated on GitHub). README screenshot at `README.md:76` is an expired
  signed `private-user-images` URL (dead image). `extra/mtgo-no-startupsound.sh` has a
  hard-coded 2020 token/version and is obsolete per commit b098029.
- **Base image runtime libs** (pauleve/docker-wine-wow64 final stage): `libglu1-mesa`
  (pulls glvnd `libgl1`), `libosmesa6`, X11 libs. **No** `libgl1-mesa-dri`, no Vulkan
  loader/drivers. NVIDIA passthrough injects driver libs via nvidia-container-toolkit
  (host has `nvidia-smi` driver 590 but **no** `nvidia-ctk` and docker lists only the
  `runc` runtime). AMD/Intel passthrough needs Mesa DRI drivers inside the image.
- **Dev host:** Debian 13, kernel 6.12, GNOME on Wayland (XWayland provides `:0`),
  docker 29.7, PipeWire-pulse socket at `/run/user/1000/pulse/native`, uid 1000.

## Upstream PR conflict map (checked 2026-09-12)

Inventory: 54 PRs on `pauleve/docker-mtgo` (3 open, 51 closed). Closed-unmerged ones
(#75, #76, #117, #123, #164, #176, #183) are all wine-version experiments or the
`hotfixes` wine-source-patch branch; nothing we propose was previously rejected. Merged
PRs we partially build on: #137 (added `renderer=gdi` + libGL to silence warnings; we keep
`gdi` as default and add GPU opt-in, same intent), #103 (`sound=alsa` default, kept),
#187 (podman FAQ, only touched if podman work proceeds).

Open PRs and the exact regions they touch:

| PR | Files / hunks | Collides with |
|----|---------------|---------------|
| #204 wine 11.2 (pauleve, 2024-08) | `Dockerfile:1` only | **Unit 1 is this PR.** Do not open a duplicate; test it, comment results, merge `origin/auto/wine-updates` in the fork. |
| #215 podman (jcastonguay, 2026-05) | `run-mtgo:6,7,11` (client default, `limit_cpus=0`, `--userns keep-id`), `:208,:219` (volume-init runs get `${opts}`) | Podman auto-detect idea → **dropped from Unit 7**; instead comment on #215 proposing auto-detect with docker default. Unit 5's clamp lives at `:295`, no textual overlap; semantic note: if #215 merges, default becomes 0 and the clamp is still correct. |
| #219 sound patch (emostar, 2026-08) | `Dockerfile:30` (COPY after `mtgo.sh`), `Makefile` sound targets, `sound/Dockerfile` (`ARG BASE`, python3), new `sound/local.Dockerfile`, `README` Sound + Troubleshooting sections, `run-mtgo:24-26` (usage), `:49` (`lopts` sound line), `:80`, `:108` (case block next to `--disable-sound`), `:133-146` (`--update`/`force_update`), `extra/mtgo.sh:2-31` (option parser + `audio_patch()`), `:85-97` (call before `run wine ${setup}` and inside the watchdog loop right after `sleep $s`) | Units 2, 3, 4, 6, 7, 8 touch the same files. Placement rules below make every hunk land on different lines, so a plain merge is clean whichever side lands first. |

Placement rules (apply in every unit):
- `run-mtgo`: new `usage()` lines go after `--bind`, never next to the sound lines; new
  options get their **own** `lopts="${lopts},…"` line appended after the `debug` one;
  new `case` entries go after `--limit-cpus`, not next to `--disable-sound`; do not touch
  the `--update`/`--)` block (#219) or lines 6-11/208/219 (#215).
- `extra/mtgo.sh`: new options are parsed in a second, separate `case` block placed after
  the WoW64 guard (line 14-22), not in the top parser; new functions are defined after
  `workaround_dotnet()`; pre-launch calls go **before** the `setup="/opt/mtgo/mtgo.exe"`
  line (#219 inserts after it); in-loop calls go inside the existing
  `started=1` branch (the `echo "====== MTGO.exe has started."` block), which #219 does not
  touch. The `winedbg` line replacement (Unit 6) is the one unavoidable adjacent hunk;
  it is a one-line resolution and is called out in that PR.
- `Dockerfile`: apt/COPY additions go in the root section before the first `USER wine`
  (lines 8-13), never near line 30.
- Do **not** touch `sound/Dockerfile`, `Makefile` sound targets, or the README Sound
  section (all owned by #219). The README Troubleshooting Ctrl+C removal (Unit 7) is an
  adjacent hunk to #219's additions there; keep it as a separate trivial commit.

Pre-PR check to run for every unit (repeatable, no working-tree side effects):
```
gh pr list -R pauleve/docker-mtgo --state open --json number,title,headRefName
for n in 204 215 219; do git fetch origin pull/$n/head:pr-$n; done
git merge-tree --write-tree master pr-219      # exit 1 + conflict listing means overlap
git merge-tree --write-tree <unit-branch> pr-219
git merge-tree --write-tree <unit-branch> pr-215
```
Re-run the inventory query (`gh pr list --state all --limit 200`) before opening each PR;
new PRs since 2026-09-12 need the same triage.

## Assumptions (override in review)
- The focus fix (`UseTakeFocus=N`) is applied **by default** (it is the standard Wine
  recommendation and cheap to revert via `--no-focus-fix`); virtual desktop stays opt-in.
- The WPF `MTGO.exe.config` flags are applied **by default** with `--no-wpf-tweaks` to
  opt out; they only touch the app's own config and are what the Proton-based setup uses.
- `--gpu` is **opt-in** (`--gpu`/`--gpu auto|nvidia|dri|none`), and the wined3d renderer
  stays `gdi` unless `--gpu` is active. Rendering glitches with WPF-over-OpenGL are a
  real possibility, so we validate before proposing a default change upstream.
- Mesa DRI/Vulkan packages go into the base image (≈150 MB) rather than a third image
  variant; simpler for users, and the maintainer already accepts a 2.3 GB image.
- Default `--limit-cpus` stays 4 (evidence in #160) but is clamped to `nproc`; raising
  it is documented, not defaulted, until measured.
- Sound (upstream PR #219) is out of scope; the plan only avoids conflicting with it.
- CI modernization is included as a low-priority unit; it only matters when sent upstream.
- Wine's native Wayland driver is **not** attempted (WPF/.NET on it is untested and the
  container path is X11 via XWayland today); listed under future work.
- esync/fsync/ntsync are not applicable: the base image is vanilla Wine, host kernel
  6.12 has no ntsync, and the Proton setup explicitly disables ntsync for MTGO.

## Approach

Ship as ordered, independent units, each validated with the A/B protocol below using
the `--test` prefix and a locally built tag. Unit 1 (wine bump) goes first so every
later measurement is against the modern base. Units 2-4 are the user-visible wins
(focus, freezes, GPU). Units 5-8 are cost/robustness/hygiene. Everything new is a flag
on `run-mtgo` that becomes an env var or argument for `extra/mtgo.sh` (existing
pattern: `--winecfg`/`--sound` are forwarded as `cmdargs`, `run-mtgo:92-99`).

Rejected alternatives: switching the base to wine-staging/Proton for esync (separate
project, contradicts upstream's own base image); a separate `:gpu` image (more CI, more
confusion than 150 MB); replacing cpuset with `--cpus` quota (.NET sizes its thread
pool from the affinity mask, which is the #160 mechanism, so cpuset is the right tool);
Wine virtual desktop as the focus default (changes UX for everyone, keep opt-in).

## Delivery process (binding for implement-plan)

Each unit below is delivered one at a time, in order, and nothing merges to `master`
without the user's explicit approval per unit:

1. **Branch per unit**: `git switch -c opt/<n>-<slug>` from `master` (after Unit 1, from
   the wine-11 base). Only that unit's files change; no drive-by fixes from other units.
2. **Build a tagged image** for the unit: `make image BASE=panard/mtgo:ab-<slug>`
   (units that change only `run-mtgo` skip the build and are tested with `--dry-run`
   plus a real run against the current image).
3. **Measure**: run the Step 0 protocol against the unit's image using the `--test`
   prefix, and put the before/after table in `plans/measurements/<slug>.md` (created on
   the branch). For units without a numeric signal (focus fix, virtual desktop, DPI,
   Ctrl+C trap, hygiene), the check is a described manual/visual scenario the user
   performs; state the exact scenario and expected outcome in that file.
4. **Stop and ask**: present diff + measurements with `AskUserQuestion` offering
   approve / rework / drop. No unit is merged, and the next unit is not started, until
   the user approves. "No measurable effect" defaults to **drop**, not merge.
5. **On approve**: run the PR conflict check from the conflict map, then merge the branch
   into `master` (fast-forward or merge commit as the user prefers) and, if the user
   says so, open the upstream PR with the measurement table in its description.
6. Units 0 and 1 are prerequisites (baseline numbers and the wine base) and follow the
   same gate: Unit 1 is approved on "MTGO still installs, logs in, and starts a match"
   plus the baseline numbers, not on a speed win.

## Implementation steps

### 0. Baseline and A/B protocol (no code)
Record before touching anything, then after each unit. Use `make image BASE=panard/mtgo:ab-<unit>`
and `./run-mtgo --test --reset panard/mtgo:ab-<unit>` so the production volume is untouched.
- Time from `./run-mtgo` to the login window (stopwatch or `date` around the run).
- CPU while idling on the Collection tab, while scrolling the collection, and after a
  right-click context menu (#149): `docker stats mtgo_running --no-stream` sampled 5×.
- Focus test: open Chat + Trade + a duel window; note whether any window grabs focus
  while typing in another for 2 minutes.
- Count of stderr lines per launch (log noise regression check).
Put the numbers in the PR description of each unit.

### 1. Bump wine to 11.2-wow64 (adopt upstream PR #204, no new PR)
- Merge `origin/auto/wine-updates` (commit 71aa347, `Dockerfile:1` →
  `FROM panard/wine:11.2-wow64`) into the fork's working branch. Upstream: post the
  test results as a comment on #204 instead of opening a duplicate.
- Build, run `--test --reset`; confirm `winetricks dotnet48` still installs under wine 11
  (winetricks is pulled from `master` at base-image build time) and that the WoW64
  guard in `mtgo.sh:14` passes. If dotnet48 fails, pin a known-good winetricks commit via
  the base image's `WINETRICKS_VERSION` ARG and note it.
- Pull `panard/mtgo:latest` too, so the "before" baseline is 9.14, not the local 9.8.

### 2. Focus-stealing fix + `--desktop` option
- `extra/mtgo.sh`: new `apply_focus_fix()` run before `wineboot`:
  `wine reg add 'HKCU\Software\Wine\X11 Driver' /v UseTakeFocus /t REG_SZ /d N /f`,
  skipped when `--no-focus-fix` is passed.
- `extra/mtgo.sh`: `--desktop WxH` → `winetricks vd=WxH`; `--no-desktop` → `winetricks vd=off`.
  (Persisted in volume `user.reg`, so passing it once is enough; document that.)
- `run-mtgo`: forward `--no-focus-fix`, `--desktop`, `--no-desktop` into `cmdargs`
  (same pattern as `--winecfg`), add to `lopts` and `usage()`.
- `README.md`: replace the "run `--winecfg` and click Graphics tab" paragraph with the
  two flags; drop the dead screenshot link; reference #196.

### 3. WPF UI-thread tweaks in `MTGO.exe.config`
- `extra/mtgo.sh`: `tune_wpf()` that locates every
  `drive_c/users/wine/**/Apps/2.0/**/MTGO.exe.config` (use `find` from the users dir,
  it is the mounted volume) and flips the four keys with one sed:
  `s/\(\(DisableAutomationPeer\|PurgeAutomationEvents\|DisableStylusInput\|DisableTabletDevices\)" value="\)false/\1true/g`.
  Idempotent; log a line when a file was changed.
- Call it right after the `workaround_dotnet` call (before the `setup=` line, per the
  placement rules) and once more inside the watchdog's `started=1` branch (MTGO
  self-updates install a fresh app dir; the next launch then picks the tweak up, and the
  in-loop call covers same-session updates).
- `--no-wpf-tweaks` opt-out forwarded from `run-mtgo`.
- Verify ClickOnce does not reject the modified config (the Proton setup does the same
  edit and PR #219 patches a DLL in the same dir, so hash checks are not enforced at
  launch; confirm on first run).

### 4. GPU passthrough (`--gpu`) and renderer switch
- `run-mtgo`: `--gpu [MODE]` with MODE `auto` (default when flag given), `nvidia`,
  `dri`, `none`. Detection in `auto`, Linux only:
  - `nvidia`: `nvidia-smi` on PATH **and** `docker info --format '{{.Runtimes}}'`
    mentions `nvidia` → `--gpus all -e NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute`.
    If `nvidia-smi` exists but the runtime is missing, print the one-time host setup
    (`sudo apt install nvidia-container-toolkit && sudo nvidia-ctk runtime configure
    --runtime=docker && sudo systemctl restart docker`) and fall through to `dri`.
  - `dri`: `/dev/dri` exists → `--device /dev/dri` plus `--group-add` for the numeric
    gids of `video` and `render` (`getent group render | cut -d: -f3`; the container's
    group names differ, so pass numbers).
  - When a mode is active, add `-e MTGO_RENDERER=gl`. macOS: `--gpu` prints "not
    supported" and continues.
- `extra/mtgo.sh`: replace the fixed verbs with
  `winetricks ${commontricks} renderer=${MTGO_RENDERER:-gdi} …` so the build-time
  `renderer=gdi` is overridden per launch (`renderer=gl` for `--gpu`; leave
  `MTGO_RENDERER=vulkan` reachable via `-e` for experiments).
- `Dockerfile`: as root, `apt-get install --no-install-recommends libgl1-mesa-dri
  mesa-vulkan-drivers libvulkan1 libegl1` then back to `USER wine`; keep the `renderer=gdi`
  build-time default. (NVIDIA needs only glvnd `libgl1`, already present via `libglu1-mesa`.)
- Verify inside `--shell --gpu`: `glxinfo -B` (add `mesa-utils` only for the check, not
  the image) or `wine` stderr no longer showing the libGL/"Direct3D9 not available" line;
  then run the A/B protocol. If WPF renders glitched under `gl`, try
  `MTGO_RENDERER=vulkan`, then leave `gdi` default and document `--gpu` as experimental.
- Optional experiment (no code unless it wins): with no GPU, set
  `HKCU\Software\Microsoft\Avalon.Graphics\DisableHWAcceleration=1` (DWORD) so WPF skips
  the failing D3D probe and uses its own software rasterizer; measure #149's right-click
  latency both ways.
- `README.md`: new "GPU acceleration" section (prereqs per vendor, the toolkit one-liner,
  what to expect), link #132/#149.

### 5. CPU limit fix
- `run-mtgo`: clamp `limit_cpus` to `nproc` (Linux) / `sysctl -n hw.ncpu` (macOS); when
  `limit_cpus >= nproc` or `0`, do not emit `--cpuset-cpus` at all. Fixes the
  <4-CPU crash.
- Optional in the same unit: on Linux pick one CPU per physical core from
  `/sys/devices/system/cpu/cpu*/topology/thread_siblings_list` before falling back to
  `0-(N-1)`, so `--limit-cpus 4` never lands on two SMT siblings.
- Document in README that `--limit-cpus 0` disables the pin and that 4 is the #160 workaround.

### 6. Cheaper launch and watchdog (`extra/mtgo.sh`)
- Stamp file `~/.wine/host/.mtgo-tricks-<sha1 of verb string + wine --version>`; run the
  `winetricks` settings line only when the stamp is missing (settings live in the
  volume-backed `user.reg`, so they survive). `--winecfg`, `--desktop`, `--sound` toggles
  still force a run.
- Replace the `winedbg --command "info proc"` poll with `pgrep -f 'MTGO\.exe'`
  (verify in `--shell` that the Wine process shows the exe path in its cmdline; if not,
  keep winedbg but raise the interval). Keeps the same start/stop state machine. This is
  the one hunk adjacent to #219's in-loop insertion; say so in the PR.
- `workaround_dotnet`: `COPY extra/mscoreei.dll /opt/mtgo/mscoreei.dll` in `Dockerfile`
  next to the existing `COPY extra/live-mtgo` (line 11), and `cp` it instead of curling
  GitHub at runtime (offline-safe, no supply-chain fetch).

### 7. `run-mtgo` robustness
- Remove `set -x` (`run-mtgo:61`); quote `"${_tz}"` (`run-mtgo:169`);
  `${docker_client} info >/dev/null || exit 1` (`run-mtgo:175`); `set -o pipefail`.
- `trap` on INT/TERM that runs `${docker_client} stop -t 5 ${name}` so Ctrl+C actually
  stops MTGO; drop the README troubleshooting entry it makes obsolete.
- Before `docker run`: if a container named `${name}` exists, say so and offer
  `--name`/`docker rm -f` instead of failing on the name clash.
- Podman: **no code here** (owned by open PR #215). Leave a review comment on #215
  suggesting auto-detection (`docker` if on PATH, else `podman`, then `--userns keep-id`)
  so it can merge without flipping the default; revisit only if #215 is closed unmerged.
- `--dpi N` → forwarded, `mtgo.sh` does `wine reg add 'HKCU\Control Panel\Desktop' /v
  LogPixels /t REG_DWORD /d N /f` (HiDPI users; persisted in the volume).

### 8. Dockerfile, repo, and CI hygiene
- `Dockerfile`: `ENV K=V` syntax, single `USER wine`, `.dockerignore` (`.git`, `plans`,
  `*.md`). `sound/Dockerfile` and the Makefile sound targets are left alone (PR #219
  already parameterizes them with `ARG BASE`).
- Delete `extra/mtgo-no-startupsound.sh` (obsolete since b098029; hard-coded token).
- Workflows: `actions/checkout@v4`, `docker/login-action@v3`, replace
  `actions-hub/docker@master` with plain `docker push` steps, `create-pull-request@v7`,
  `pascalgn/automerge-action` latest. Keep the tag/label flow unchanged.
- README: GPU section, `--desktop`/`--dpi`/`--gpu`/`--limit-cpus 0` docs, remove the
  dead screenshot and the Ctrl+C entry. Do not touch the Sound section (#219) or the
  podman FAQ (#215).

## Testing
- Every unit: `make image BASE=panard/mtgo:ab-<unit>` then
  `./run-mtgo --test --reset panard/mtgo:ab-<unit>` (fresh prefix) **and**
  `./run-mtgo --test panard/mtgo:ab-<unit>` (existing prefix) — the second run covers the
  stamp/idempotency paths.
- Run the Step 0 protocol after units 1, 3, 4, 5 and put the table in each PR.
- Unit 2: the focus test from Step 0, plus `--desktop 1920x1080` then `--no-desktop` and
  confirm `user.reg` in the volume flips `Explorer\Desktops`.
- Unit 4: `./run-mtgo --dry-run --gpu` prints `--gpus all …` on this host; after
  installing the toolkit, `--shell --gpu` shows the NVIDIA GLX renderer; `--gpu dri`
  path exercised by forcing it (`--gpu dri`) on the same host (Raphael iGPU present).
- Unit 5: `./run-mtgo --dry-run --limit-cpus 64` shows no `--cpuset-cpus`;
  `--limit-cpus 2` shows `0-1` (or SMT-aware picks).
- Unit 7: Ctrl+C during a run leaves no `mtgo_running` container (`docker ps -a`);
  `bash -n run-mtgo` and `shellcheck run-mtgo extra/mtgo.sh` clean or with justified
  suppressions; macOS branch reviewed by reading only (no macOS host available).
- Unit 8: `docker build` emits no legacy-ENV warnings; `act` or a PR on the fork to
  confirm workflows parse.
- Every unit before opening a PR: the `git merge-tree` checks from the conflict map
  against `pr-204`, `pr-215`, `pr-219` return clean (only Unit 6's `winedbg` line is
  allowed to report a conflict, and the PR text must mention it).

## Risks / open questions
- **dotnet48 on wine 11** may need a specific winetricks revision; if it breaks, Unit 1
  stalls everything and should be reported upstream on PR #204.
- **WPF over real OpenGL** can render wrong (fonts, hover effects) under wined3d; that is
  why `gdi` stays the default. Result of Unit 4's A/B decides whether to argue for a
  default change upstream.
- **ClickOnce config edits** (Unit 3): confirm the client does not re-validate the config
  hash at launch; if it does, fall back to editing `Shiny...user.config` or drop the unit.
- **NVIDIA on Debian 13 host**: nvidia-container-toolkit is a host-side install the user
  must do; the script can only detect and instruct.
- **cpuset default**: #160's cause was never fully diagnosed; keep 4 until Step 0 numbers
  show more cores help rather than hurt.
- **Focus fix side effects**: `UseTakeFocus=N` changes how Wine re-acquires focus after
  Alt-Tab; the Wine bug proposes it as the Linux default, but confirm no regressions in
  duel windows.
- **Future, not planned**: Wine native Wayland driver (mount `$XDG_RUNTIME_DIR/wayland-0`,
  unset `DISPLAY`) once WPF apps are known-good on it; sound via PR #219; DXVK
  (`winetricks dxvk`) on top of `--gpu` for D3D9→Vulkan.

## Hand-off
After approval, copy this file to `plans/mtgo-optimizations.md` in the repo and run
`/implement-plan plans/mtgo-optimizations.md`. implement-plan must follow the
"Delivery process" section: one branch per unit, measure, ask, then merge or drop.
