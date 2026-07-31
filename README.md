# MTGNP v1.0 — Magic: The Gathering Network Protocol

CSNETWK Machine Problem — implementation of RFC 0001 (MTGNP v1.0).

> **NOTE:** the rubric requires the README as a **PDF**. Convert this file
> before submission and fill in every `TODO` (names, work matrix, AI usage).

## Project structure

```
project/
├── server/                     # Game Server package — run: python3 -m server.main
│   ├── main.py                 #   entry point, socket accept loop, lifecycle (Dev 1)
│   ├── transport.py            #   framing, PDU codec, ClientConn, dispatch table (Dev 1)
│   ├── heartbeat.py            #   PING/PONG (Dev 1)
│   ├── lobby.py                #   PLAYER_READY handling, LOBBY/SETUP/MULLIGAN (Dev 2)
│   ├── phase_engine.py         #   phase/step machine, PHASE_TRANSITION, land drops (Dev 2)
│   ├── combat.py               #   attackers/blockers/damage order/first strike (Dev 4)
│   ├── card_catalog.py         #   loads shared JSON catalog; ILLEGAL_DECK validation (Dev 3)
│   ├── game_state.py           #   authoritative data model + GameOver/ClientGone (Dev 3)
│   ├── state_view.py           #   personalized GAME_STATE_UPDATE views, hidden hands (Dev 3)
│   ├── priority.py             #   PRIORITY_GRANT/PASS, seq_num tokens, STALE_ACTION (Dev 3)
│   ├── stack.py                #   LIFO stack, STACK_PUSH/RESOLVE, SBAs, triggers (Dev 3)
│   ├── effects.py              #   card effects, mana payment, INSUFFICIENT_MANA (Dev 3)
│   └── tests/
│       ├── test_card_catalog.py       # unit: catalog + deck validation
│       ├── test_game_state.py         # unit: data model + hidden info
│       ├── test_priority_stack.py     # unit: tokens, LIFO stack, counterspell
│       ├── test_effects.py            # unit: mana, effects, state-based actions
│       ├── test_integration_game.py   # end-to-end protocol game   (10 assertions)
│       ├── test_integration_combat.py # combat/effect mechanics     (3 assertions)
│       └── test_integration_edge.py   # edge cases + TRIGGER_ORDER (18 assertions)
├── shared/
│   └── cards.json              # shared card catalog (out-of-band card data, RFC §1)
├── client/
│   └── main.py                 # Player Client & rendering — run: python3 -m client.main (Dev 4)
├── decks/                      # sample deck lists (one card instance ID per line)
├── requirements.txt            # states that only the Python stdlib is required
├── README.md                   # this document (Markdown source)
└── README.pdf                  # this document (submitted deliverable)
```

The server is a Python package: one module per protocol concern, composed
into the `Server` class in `server/main.py` via mixins. Method
implementations are identical to the reference single-file build; only the
file layout differs.

Requirements: **Python 3.10+**, standard library only. No third-party packages.

## Running — step by step

**Prerequisites:** Python 3.10 or newer. No installation of packages is
needed (standard library only).

> **Windows note:** the command is `python` (or `py`), not `python3` —
> substitute it in every command below, e.g. `python server.py --verbose`.
> If Windows says *"Python was not found"*, install Python from
> python.org/downloads and tick **"Add python.exe to PATH"** in the
> installer, then reopen the terminal. Check with `python --version`
> (Linux/macOS: `python3 --version`).

1. **Unzip the project** and open a terminal in the project folder (the one containing the `server/` package).
2. **Start the server** (Terminal 1). It listens on port 4444 (RFC 5.1):
   ```
   python3 -m server.main --verbose
   ```
   You should see `[server] listening on port 4444`. Use `--port <n>` if
   4444 is taken (then add `--port <n>` to the clients too).
3. **Start the first client** (Terminal 2):
   ```
   python3 -m client.main --id player_1 --deck decks/burn.txt --verbose
   ```
   If the server is on another machine, add `--host <server-ip>`.
4. **Start the second client** (Terminal 3):
   ```
   python3 -m client.main --id player_2 --deck decks/control.txt --verbose
   ```
   A third connection attempt will be refused by the server (RFC 5.1).
5. **Join the game:** type `ready` in each client and press Enter. When both
   are ready the server deals hands and the mulligan begins.
6. **Mulligan:** type `keep` to keep your hand, or `mull` to redraw. After
   N mulligans, keep with `keep <card_1> ... <card_N>` to put N cards on
   the bottom of your library (London mulligan).
7. **Play:** when you see `>>> You have priority`, you may act:
   * `play mountain_003` — play a land (your main phase only, 1/turn)
   * `cast lightning_bolt_001 player_2` — cast a spell (mana is paid
     automatically from your untapped lands)
   * `pass` — pass priority
   * `attack goblin_guide_001` — declare attackers (in DECLARE_ATTACKERS)
   * `block wall_of_stone_001:goblin_guide_001` — declare blockers
   * `order <attacker> <blocker1> <blocker2>` — damage order (multi-blocks)
   * `discard <card>` — discard to 7 at cleanup; `yes` / `no` — answer an
     optional trigger; `hand` / `state` — re-print; `concede` — give up;
     `help` — full list
8. **Game over:** the winner and reason are announced; both clients stay
   connected and can type `ready` to start a new game (RFC 6.6).

### Verbose mode (rubric prerequisite)

`--verbose` / `-v` on **either program** prints **every PDU sent and
received**, labelled with direction, peer, timestamp, and the full JSON:

```
[14:03:22] SEND  player_1 | {"type": "PRIORITY_GRANT", "player_id": "player_1", ...}
[14:03:23] RECV  player_1 | {"type": "CAST_SPELL", "seq_num": 12, ...}
```

Without the flag both programs run quietly. This satisfies the "Verbose Mode
Requirement" toggle in the project specification.

### Debug flags (server)

* `--time-limit <ms>` — priority deadline (default 60000; RFC 4.2)
* `--seed <n>` — seed the RNG for reproducible shuffles/coin flips (demos)
* `--first {0,1}` — force which seat goes first (demos/tests)
* `--port <n>` — listen port

### Tests

```
python3 server/tests/test_card_catalog.py     # unit tests (catalog/decks)
python3 server/tests/test_game_state.py       # unit tests (data model)
python3 server/tests/test_priority_stack.py   # unit tests (tokens/stack)
python3 server/tests/test_effects.py          # unit tests (effects/mana/SBAs)
python3 server/tests/test_integration_game.py    # 10 protocol assertions
python3 server/tests/test_integration_combat.py  #  3 combat assertions
python3 server/tests/test_integration_edge.py    # 18 edge-case assertions
```

All are run from the project root. `test_integration_edge.py` derives a deterministic RNG seed by replicating the server's
seeded shuffle locally, so its scripted 10-turn game is fully reproducible.

## Design summary

* **Transport (RFC 5):** every PDU is a 4-byte big-endian length prefix +
  UTF-8 JSON, max 65,535 bytes. `common.recv_pdu` reads exactly the framed
  bytes before parsing; malformed payloads yield `ERROR INVALID_JSON`.
* **Threads:** the server runs one reader thread per client feeding a single
  event queue; the main thread runs the RFC Section 6 lifecycle
  (`LOBBY → GAME_SETUP → MULLIGAN → IN_GAME → GAME_OVER → LOBBY`, same TCP
  connections). `PING` is answered with `PONG` directly in the reader thread
  (independent heartbeat counter, RFC 5.4). The client runs reader,
  heartbeat (PING/30 s, 10 s PONG deadline) and stdin threads.
* **Authority (RFC 4.2/4.3):** all rules live in the server. The client never
  simulates outcomes; each personalized `GAME_STATE_UPDATE` (own hand only,
  opponent hand as a count) overwrites its view.
* **seq_num discipline (RFC 5.4):** the server keeps one monotonically
  increasing counter; a broadcast consumes one number. The client echoes the
  seq of the latest `PRIORITY_GRANT` for priority actions, the relevant
  request PDU for `MULLIGAN_CHOICE`/`DISCARD` (the `GAME_STATE_UPDATE`) and
  combat declarations (the `PHASE_TRANSITION`), and any last server seq for
  `CONCEDE`. Mismatches get `ERROR STALE_ACTION` and, when the player still
  holds priority, a re-granted token. After an *illegal* (non-stale) action
  the server re-issues `PRIORITY_GRANT` with the **same** seq_num
  (RFC Sec. 11 item 3).
* **Rules implemented:** London mulligan; full phase sequence with
  first-turn draw skip; one land per turn at sorcery speed; implicit mana
  (declared `mana_payment` validated against untapped lands, which are then
  tapped); LIFO stack with priority windows and caster-retains-priority;
  fizzle on illegal targets at resolution; state-based actions after every
  event (lethal damage, 0 toughness, life ≤ 0, AP loses simultaneous-death
  ties); summoning sickness; combat with attack tapping, single-attacker
  blocks, multi-block damage ordering (`ASSIGN_DAMAGE_ORDER`), first-strike
  step when relevant, no trample; cleanup discard-to-7 loop; win by
  `LIFE_ZERO`, `DECK_EMPTY`, `CONCEDE`, `DISCONNECT` (incl. priority
  timeout); extra connections refused; lobby deck resubmission; duplicate ID
  rejection.
* **Card effects (≥ 5, RFC Appendix set):** Lightning Bolt / Shock (damage),
  Counterspell (counters a stack item), Giant Growth (+3/+3 until EOT),
  Healing Salve (lifegain), Divination (draw 2), Gray Merchant of Asphodel
  (ETB drain-2 optional **triggered ability** exercised through
  `TRIGGER_CHOICE` / `TRIGGER_CHOICE_RESPONSE`), Festering Imp (death
  trigger; two dying simultaneously exercises `TRIGGER_ORDER` /
  `TRIGGER_ORDER_RESPONSE`), Goblin Guide (haste),
  Youthful Knight (first strike), plus vanilla creatures and five basic
  lands.

## Known deviations / interpretations

* A countered spell is announced with `STACK_RESOLVE` `result: "FIZZLE"`
  (the RFC only defines `RESOLVED | FIZZLE`, with no dedicated "countered"
  result).
* Gray Merchant's drain is modelled as an *optional* ("you may") trigger so
  the `TRIGGER_CHOICE` flow of RFC 8.6.3 is exercised; declining discards it.
* `ACTIVATE_ABILITY` is answered with `ILLEGAL_ACTION`: the fixed card set
  defines no activated non-mana abilities, and mana is implicit (RFC 7.5).
  As a result the interactive client has no command that sends it (the
  server-side rejection path is still fully implemented).
* Summoning sickness is cleared for the active player's creatures during
  their Cleanup (equivalent, for this card set, to "since your last turn
  began").
* `--seed` / `--first` are non-RFC debug conveniences.

## Work Distribution Matrix (TODO — fill in)

| Member | Tasks | % |
|---|---|---|
| TODO Name 1 | e.g. framing + lobby + mulligan | TODO |
| TODO Name 2 | e.g. turn engine + priority/stack | TODO |
| TODO Name 3 | e.g. combat + SBAs + triggers | TODO |
| TODO Name 4 | e.g. client + tests + README | TODO |

## AI Usage Disclosure (TODO — edit truthfully)

TODO: state which AI tools were used (e.g. *Claude was used to generate an
initial implementation of the server/client and the test scripts; all
members reviewed, tested, and can explain every part of the code*), what
they were used for, and what was written/modified by hand. Every member must
be able to explain all of the code at the demo — undisclosed AI use or
inability to explain the code is treated as academic dishonesty by the
rubric.
