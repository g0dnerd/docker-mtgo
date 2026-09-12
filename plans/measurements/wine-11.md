# Unit 1: wine 9.14 → 11.2 (upstream PR #204) + installer launch fix

Branch: `opt/1-wine-11`. Image: `panard/mtgo:ab-wine11` (built from the branch).
Before image: `panard/mtgo:ab-baseline` (master, wine 9.14).

## What changed

1. `Dockerfile`: `FROM panard/wine:11.2-wow64` (cherry-pick of upstream 71aa347, PR #204).
2. `extra/mtgo.sh`: the installer is copied into `drive_c` and started as
   `C:\mtgo-setup.exe` instead of `/opt/mtgo/mtgo.exe`.

## Why the second change is required

With wine 11.2, starting *any* executable by its unix path segfaults in the wine
loader when the `Z:` drive mapping is absent. `extra/mtgo.sh` removes `Z:` on
purpose (commit 1f99d2b, issues #191/#155), so the stock launcher dies at
`run wine /opt/mtgo/mtgo.exe`:

```
/usr/local/bin/mtgo: line 26:   852 Segmentation fault      (core dumped) "${@}"
```

Reproduction inside the container (fresh `--test --reset` prefix):

| command | Z: present | result |
|---------|-----------|--------|
| `wine /tmp/cmd32.exe /c echo hi` | no  | SIGSEGV (exit 139) |
| `wine /tmp/cmd64.exe /c echo hi` | no  | SIGSEGV (exit 139) |
| `wine /tmp/cmd32.exe /c echo hi` | yes | `hi` |
| `wine C:\setup.exe` (copy of mtgo.exe) | no | installer runs, MTGO reaches login |

## Verification done by the implementer

- `winetricks dotnet48` installs under wine 11.2 with current winetricks master
  (no pin of `WINETRICKS_VERSION` needed); `mscoreei.dll` and `syswow64` present.
- Fresh prefix run (`./run-mtgo --test --reset panard/mtgo:ab-wine11`) reaches
  the MTGO login window after the fix; `MTGO.exe` runs at ~1.8 GB RSS.
- Registry inside the running container: `win7` and `sound=alsa` applied,
  `DllOverrides` = gdiplus/mscoree/winegstreamer/wmp as set by the launcher.
- `git merge-tree` against `pr-204`, `pr-215`, `pr-219`: clean.

## Side finding (affects Unit 4, no change here)

Neither image contains `HKCU\Software\Wine\Direct3D` — the Dockerfile's
`winetricks renderer=gdi` (and its `sound=alsa`) never persist into the image's
`user.reg`; the registry file predates the last winetricks calls in the build
layer. `sound=alsa` is re-applied by the launcher at every start, `renderer` is
not, so MTGO currently runs on wined3d's *default* renderer (OpenGL) in a
container with no `libGL.so.1`, not on `gdi` as the plan assumed. Unit 4 must
treat "renderer unset" as the real baseline.

The wine 11.2 base image also has no `libgl1`/`libglx0` (only `libglvnd0`,
`libopengl0`), so GPU passthrough needs those in addition to Mesa DRI drivers.

## Results (user-run, fill in before approving)

| image | start→login (fresh) | start→login (2nd run) | idle CPU % | scroll CPU % | right-click CPU % | focus stolen? | stderr lines |
|-------|---------------------|-----------------------|------------|--------------|-------------------|---------------|--------------|
| ab-baseline (9.14) | | | | | | | |
| ab-wine11 (11.2)   | n/a (stock launcher segfaulted; manual `wine C:\setup.exe` used) | see below | 2.0 / 2.1 / 2.2 / 5.9 / 1.8 | 212 / 257 / 3 / 154 / 180 | 9.1 / 4.9 / 6.5 / 4.9 / 5.8 | none observed (2 min typing in Chat; event-opening case not testable without tickets) | 15 err/fixme/warn lines |

Memory during the session: 2.9 GiB idle, 3.4-3.8 GiB scrolling, 3.9 GiB after
the context menu. CPU % is `docker stats` (100 % = one core; container is
pinned to 4 cores). The right-click case does *not* peg the CPU under wine 11.2
on this host, unlike the report in upstream #149.

Measured 2026-09-12 by the user (scenarios) and the implementer (`docker stats`
sampling, 5 samples 10 s apart).

Gate criterion (from the plan): MTGO installs, logs in, and starts a match on
`ab-wine11`; numbers are for reference, not a speed requirement.

## Draft comment for upstream PR #204 (not posted)

> Tested `panard/wine:11.2-wow64` on Debian 13 / docker 29.7 (Ryzen 7700X, X11 via
> XWayland). `winetricks dotnet48` still installs with current winetricks, and
> MTGO installs and reaches the login window. One change is needed on top of
> the bump: with wine 11 the launcher segfaults at `run wine /opt/mtgo/mtgo.exe`
> because `extra/mtgo.sh` removes the `Z:` drive (1f99d2b) and wine 11 can no
> longer start an exe by unix path without it (`wine /tmp/x.exe` → SIGSEGV,
> `wine C:\x.exe` fine). Copying the installer into `drive_c` and starting it as
> `C:\mtgo-setup.exe` fixes it; happy to send that as a follow-up PR or you can
> fold it into this one.
