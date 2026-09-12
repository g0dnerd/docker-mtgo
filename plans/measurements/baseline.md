# Unit 0: baseline and A/B protocol

Images used for every A/B in this series:

| tag | built from | wine |
|-----|------------|------|
| `panard/mtgo:ab-baseline` | `master` (2e2afb7) | 9.14-wow64 |
| `panard/mtgo:ab-<unit>` | the unit's branch | 11.2-wow64 (from Unit 1 on) |

Docker Hub `panard/mtgo:latest` was last pushed 2024-05-06, before the 9.14 bump,
so it is *not* used as the "before" image; `ab-baseline` is built locally from
`master` instead.

## Protocol (run per image, always with `--test` so the production volume is untouched)

```
# fresh prefix (installer + .NET checks), stop the clock at the login window
date +%T; ./run-mtgo --test --reset panard/mtgo:<tag> 2>stderr.log; date +%T
# second run against the existing prefix (idempotency / stamp paths)
date +%T; ./run-mtgo --test panard/mtgo:<tag> 2>stderr.log; date +%T
```

While MTGO is running, in another terminal sample CPU five times (~10 s apart):

```
for i in 1 2 3 4 5; do docker stats mtgo_running --no-stream --format '{{.CPUPerc}} {{.MemUsage}}'; sleep 10; done
```

Scenarios:
1. **Idle** on the Collection tab, no mouse movement.
2. **Scrolling** the collection continuously.
3. **Right-click** a card so the context menu is open (upstream #149).
4. **Focus**: open Chat, Trade and a duel/practice window; type in Chat for
   2 minutes. Record whether any other window took focus or moved on its own.
5. **Noise**: `wc -l stderr.log` from the fresh-prefix run.

## Results

| image | start→login (fresh) | start→login (2nd run) | idle CPU % | scroll CPU % | right-click CPU % | focus stolen? | stderr lines |
|-------|---------------------|-----------------------|------------|--------------|-------------------|---------------|--------------|
| ab-baseline (9.14) | | | | | | | |

Fill in the row above before approving Unit 1; every later unit adds a row to
its own `plans/measurements/<slug>.md`.
