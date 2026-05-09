# Detailed analysis of `kidschess/chessjs`

## Short version

`chessjs` is a **JavaScript port of "VICE"** (Bluefever Software's well-known C tutorial chess engine), wrapped in a minimal jQuery + HTML UI and hosted on GitHub Pages as "Kindle Chess". It's a small project (35 commits, ~600 KB total, default branch `develop`) whose engine source is essentially a faithful translation of Richard Allbert's tutorial code into Javascript. 

## Architecture overview

`index.html` loads twelve scripts in dependency order, all running on the **main thread** (no Web Worker, no WebAssembly):

| File | Purpose |
|---|---|
| `defs.js` | Constants, piece encoding, square mapping, bit-packed move format, Zobrist key tables, the `GameController` global |
| `io.js` | FEN parsing, algebraic move printing |
| `board.js` | Board reset, FEN load, position-key recompute, the (dormant) opening-book matcher |
| `movegen.js` | `GenerateMoves` (all moves) and `GenerateCaptures` (quiescence-only) |
| `makemove.js` | `MakeMove` / `TakeMove` with incremental Zobrist updates and undo-history stack |
| `perft.js` | A `perft` (performance-test) move-counter — used to validate move generation against known node counts |
| `evaluate.js` | Static evaluation: material + piece-square tables + a few pawn-structure terms |
| `pvtable.js` | Principal-variation hash table (NOT a real transposition table — see below) |
| `search.js` | Alpha-beta + quiescence + null-move + iterative deepening, plus DOM updates |
| `protocol.js` | Glue between UI buttons and the engine |
| `gui.js` | jQuery-driven board rendering, drag-and-drop, click-to-move |
| `main.js` | Boot code; the opening-book AJAX loader is **commented out** here |

Plus jQuery 1.10.1 (from 2013), which is hilariously dated — it's used essentially just for `$.now()`, DOM manipulation, and an `ajaxComplete` hook that does nothing.

## Board representation

Standard textbook **10×12 mailbox**: a 1-D array `BRD_SQ_NUM = 120`, where the playing squares occupy indices 21–98 and the rest are `OFFBOARD` sentinels. This makes off-board detection a single array lookup (`SQOFFBOARD`) rather than coordinate arithmetic, which is a classic CPU-friendly trade-off (and a sensible one in JS too, since modulo and double-comparison branches are slow).

Two parallel mappings, `Sq120ToSq64` and `Sq64ToSq120`, convert to a 0–63 index when needed (e.g. for piece-square table lookups). Move directions are flat integers (`KnDir = [-8,-19,-21,-12,8,19,21,12]`, etc.), so a bishop sliding north-east just keeps adding `+11` until it hits something.

A separate **piece list** (`brd_pList`, indexed via the macro `PCEINDEX(pce, n) = pce*10 + n`) is maintained alongside the mailbox, so the evaluator can iterate over "all white knights" without scanning all 64 squares — the typical VICE optimisation.

## Move encoding

Each move is a single 25-bit integer, decoded by the macros at the bottom of `defs.js`:

```
bits  0–6   from-square  (FROMSQ)
bits  7–13  to-square    (TOSQ)
bits 14–17  captured piece (CAPTURED)
bit  18     en-passant flag (MFLAGEP)
bit  19     pawn-double-push flag (MFLAGPS)
bits 20–23  promoted piece (PROMOTED)
bit  24     castle flag (MFLAGCA)
```

Plus the fast-test masks `MFLAGCAP = 0x7C000` (any-capture, including EP) and `MFLAGPROM = 0xF00000` (any-promotion). This is identical to VICE's encoding.

## Search

`SearchPosition()` runs **iterative deepening** from depth 1 upward, calling `AlphaBeta` at each depth, with a wall-clock cutoff (`srch_time`, settable to 1/2/4/6/8/10 s in the UI). The CheckUp call only fires every 2048 nodes (`(srch_nodes & 2047) == 0`) — which is fine on modern hardware but explains why "1 second" thinking can overshoot noticeably on slow machines.

`AlphaBeta` is a textbook **negamax fail-hard alpha-beta** with these enhancements:

- **Quiescence search** at the leaves (`Quiescence`), expanding only captures via `GenerateCaptures`, with a stand-pat cutoff. No SEE pruning, no delta pruning.
- **Null-move pruning** with a fixed depth reduction `R = 3` (`depth - 4` argument with the `-1` from the recursive call), guarded by the standard "not in check, not in zugzwang-prone endgame, depth ≥ 4, side has more than ~K+pawn material". The threshold `brd_material[brd_side] > 50200` means strictly more than a king + 2 pawns of material — slightly conservative.
- **Check extension**: if the side to move is in check, depth is bumped by 1 before recursing.
- **PV-move ordering**: the previous iteration's best move (probed from `brd_PvTable`) is given a score of 2,000,000 so `PickNextMove` selects it first.
- **MVV-LVA** for capture ordering (defined in `movegen.js`).
- **Killer moves**: two slots per ply (`brd_searchKillers`, sized `3 * MAXDEPTH`).
- **History heuristic**: per `(piece, to-square)` pair, incremented by `depth` whenever a quiet move improves alpha.
- **Move ordering via selection sort**: `PickNextMove` does a single-pass max-find inside the move list and swaps it forward — cheaper than full sorting since alpha-beta usually only needs the first few moves anyway.
- **50-move rule** (`brd_fiftyMove >= 100` — 100 plies = 50 moves) and **threefold-style repetition** detection scanning the history stack.
- **Mate distance** is encoded as `-MATE + brd_ply` so the engine prefers shorter mates and longer survivals.

The `Ordering` percentage shown in the UI is `srch_fhf / srch_fh * 100` — i.e. **fraction of beta cutoffs achieved on the very first move tried**. Well-tuned engines hit > 90 %, and on a position you load up here you'll typically see 85–95 %, which is consistent with a working PV/killer/history setup.

### What's *missing* compared to a serious engine

- **No real transposition table.** `pvtable.js` is a PV-only table: it stores `(posKey, move)` pairs and is probed *only* to seed move ordering. It does not store scores or depth, so positions reached by transposition aren't re-used to cut off; only the move ordering benefits. This is a significant strength penalty.
- **No aspiration windows** — every iteration searches the full `(-INFINITE, +INFINITE)` window.
- **No late-move reductions, no futility pruning, no SEE.**
- **`EvalPosition(pos)` is called with an argument** at line 172 that the function ignores (the function takes no parameters). Harmless, but a port-from-C artifact.
- **Search runs on the main thread** with only every-2048-node abort checks, so the page genuinely freezes during a multi-second search. Putting the engine in a Web Worker would be the single biggest UX improvement.

## Evaluation

`EvalPosition()` is a side-to-move-relative score in centipawns, computed as:

1. Material balance from the incrementally-maintained `brd_material[]`.
2. **Insufficient-material draw recogniser** (`MaterialDraw`) for KNN-vs-K, KB-vs-KB-same-colour, KR-vs-KR with at most one minor each, etc. Returns 0 for those.
3. **Piece-square tables** (PST) for P, N, B, R, plus *the rook table reused for the queen* — a deliberate VICE shortcut. Black uses the white tables via `MIRROR64`.
4. **Pawn structure**: isolated-pawn penalty (−10), passed-pawn bonus indexed by rank (`[0, 5, 10, 20, 35, 60, 100, 200]` — note the `+200` at the 7th rank pre-promotion). Implemented via two single-pass arrays `PawnRanksWhite/Black[file]` containing the most-advanced own pawn on each file (with the `file ± 1` neighbour-checks padded by sentinels at index 0 and 9).
5. **Open-file bonuses** for rooks (10/5 open/semi-open) and queens (5/3).
6. **Bishop pair**: +30.
7. **King PST switching**: a "middlegame" table `KingO` that strongly penalises a centralised king, vs. an "endgame" table `KingE` that rewards centralisation. The switch is triggered when *the opponent* is below `ENDGAME_MAT = R + 2N + 2P + K = 50000 + 550 + 650 + 200 = 51400` of material — i.e. when the opponent can no longer mount a serious mating attack. A simple but effective tapered eval.

Overall this is a **roughly 1500–1800 ELO** evaluation — adequate for a casual opponent but obviously nowhere near competitive engines.

## UI / front-end

- jQuery 1.10.1 only.
- The board is rendered as plain `<img>` tags inside a `<div id="Board">`, swapped on each move. No animation, no SVG, no canvas.
- Google Analytics tag (`G-Q94743BW9M`) is wired in.

## The "Loading book, please wait…" message

This is the most visible quirk of the live page, and the answer is straightforward: **the opening book is dead code**. In `main.js`, the `$.ajax(...)` call that would download `bookXml.xml`, parse it into `brd_bookLines`, set `GameController.BookLoaded = TRUE`, and remove the loading span is fully commented out. The span is therefore never removed, and `BookMove()` is never called from `SearchPosition()` because the `BookLoaded` flag stays false. The engine plays from move 1 by raw search.

The book format itself, judging from the surviving `BookMove()` and `LineMatch()` code, was going to be a list of strings of the form `"<game-history-prefix> <next-move>"`, with a magic-number offset (`lengthOfLineHack`) to slice out the move — clearly an ad-hoc text format rather than Polyglot.

## Bottom line

This is an **educational engine** — almost certainly a student's port of the VICE tutorial, finished enough to play but with rough edges (no TT scores, dormant book, main-thread search). On the strength axis you'd likely see something around club level; on the engineering axis it's a clean, readable example of how the classical chess-engine architecture maps to JavaScript. If you wanted to improve it further, the highest-leverage changes — in roughly this order — would be:

1. Move the engine into a Web Worker so the UI doesn't freeze.
2. Promote `pvtable.js` from a PV-only cache to a real transposition table that stores `(score, depth, flag, bestMove)` and produces alpha/beta cutoffs.
3. Add aspiration windows and late-move reductions.
4. Enable an actual opening book — Polyglot `.bin` parsing in JS is a few hundred lines, or just JSON-encode a simple book.
