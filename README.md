# imgame

**imgame** is a bot for the Builders Buildathon 2026 **Mech Royale** track
(`mechfight/1`). It is a single statically linked Go binary with no third-party
dependencies. It reads observations as JSON lines on stdin and writes one action
per decision to stdout.

The match is a 16-mech free-for-all. Score is `16 - place`, so every point is a
mech that died before us. The practice lobby is a 15-bot squad that shares vision
and never fights itself: points have to be earned by killing, not by waiting.

## How it plays

- **Opening.** Arms up quickly, takes a laser when one is available, and avoids
  waking the whole squad before it has a weapon.
- **Economy.** Collects cubes, crates, and uncontested supply drops, then spends
  upgrade offers on damage, armour, and survivability. Every rule value comes
  from `hello.config`; nothing about the balance patch is hard-coded.
- **Fights.** Takes favourable duels, declines losing ones, and uses the gun plus
  a secondary weapon at the range where each is strongest. The gun and secondary
  fire together. Aim is led with an online per-opponent correction, and the
  mortar is solved ballistically when line of sight is blocked.
- **Movement.** A forward-simulated trajectory planner scores candidate moves
  against incoming bullets, mortar arcs, charging lasers, walls, and the zone. It
  also handles rooftop access and jump-jet travel.
- **Survival.** Breaks line of sight, kites multiple attackers, respects the
  shrink timer, and keeps clear of lightning carriers. The final circle is held
  more tightly as it shrinks.
- **Free-for-all.** Detects a real FFA from enemy-on-enemy fire, then switches
  from squad-farming behaviour to a more cautious brawl stance.

The bot treats a crash as a lost match, not a disqualification: malformed input
is sanitised, queued observations are skipped to the newest one, and the decision
path is wrapped in a panic guard that repeats the last safe action.

## Layout

| Path | Role |
| --- | --- |
| `cmd/imgame/main.go` | Protocol loop, newest-observation queue, JSON recovery, panic guard, `BOT_PARAMS` overrides |
| `internal/proto/proto.go` | `mechfight/1` message types and allocation-free action encoding |
| `internal/world/world.go` | Bilinear terrain, polygon-prism raycasting, line of sight |
| `internal/world/nav.go` | Nav grid, bounded A*, connected components, path queries |
| `internal/world/roofs.go` | Roof and access-point map |
| `internal/world/surface.go` | Overland surface search |
| `internal/world/trees.go` | Tree and canopy handling |
| `internal/world/exposure.go` | Visibility and exposure map |
| `internal/bot/bot.go` | Parameters, observation memory, per-tick state |
| `internal/bot/policy.go` | Main decision loop |
| `internal/bot/assessment.go` | Zone, threat, and loot assessment |
| `internal/bot/stance.go` | Lobby belief and stance selection |
| `internal/bot/movement.go` | Trajectory planner, dodging, unsticking |
| `internal/bot/combat.go` | Aiming, ballistics, weapon dispatch |
| `internal/bot/aim_lead.go` | Online adaptive lead |
| `internal/bot/fight_plan.go` | Range-aware fight plan and exploration |
| `internal/bot/cover.go` | Cover search |
| `internal/bot/laser_peek.go` | Laser corner-peeking |
| `internal/bot/rooftops.go` | Roof climbing, holding, and overland travel |
| `internal/bot/hideouts.go` | Embedded visibility map for hideout selection |
| `internal/bot/geometry.go` | Shared geometry helpers |

## Build and check

The repository root is the Docker build context. The arena is supplied by the
server at `/arena` and is not bundled in the image. Download the official bot kit
from the track page and point `KIT` at it.

```sh
KIT=$(realpath ../mechfight-kit)

docker build --platform linux/amd64 -t imgame .

python3 "$KIT/check/check.py" --map-dir /arena -- \
  docker run --rm -i --platform linux/amd64 --network none --cpus 1 \
  --memory 512m --memory-swap 512m --read-only --tmpfs /tmp:rw,size=64m \
  --pids-limit 128 --cap-drop ALL --user 65534:65534 \
  -v "$KIT/arena:/arena:ro" imgame
```

To run without a container:

```sh
python3 "$KIT/check/check.py" -- go run ./cmd/imgame
```

`check.py` verifies protocol conformance and the 50 ms decision budget. It
replays recorded observations; it is not a game simulation.

### Verification

The container above passed the official conformance check with **300/300**
on-time replies, zero dropped or stale ticks, and zero malformed lines:

| Metric | Value |
| --- | --- |
| Ready time | 0.6 s |
| Decision p50 | 0.6 ms |
| Decision p99 | 2.1 ms |
| Decision max | 2.6 ms |
| Image size | 4.05 MB (`FROM scratch`) |

## Results

Local measurements use a separate practice simulator and are not official
scores. The final Go build averaged **4.70** over 192 practice matches, split
evenly across two house-bot configurations, and **8.35 ± 0.38** over 24
16-mech FFA matches, winning 13 of 24. An earlier Python version of this bot
reached **5.33** on the official practice board.

## Notes

- Go 1.25, standard library only. The build is pinned to `linux/amd64` and runs
  as uid 65534.
- The official protocol, arena format, and conformance fixtures belong to the
  organiser. No kit files or third-party bot code are included in this
  repository or the image.
- `BOT_PARAMS` accepts a JSON object that overrides the default parameters for
  local experiments. Production runs with no environment overrides.
- Released under the MIT License; see [`LICENSE`](LICENSE).
