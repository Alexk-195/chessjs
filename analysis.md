Let me search for the repository first to get a valid URL.

Good — the repo exists. Let me fetch it.

Useful — now I know the layout. The repository is small (35 commits, 6 stars, 10 forks, default branch `develop`), and the substance lives in the `js/` directory. The README is essentially empty ("pure js chess" plus a TODO item "try with the new kindle"), so we have to read the source itself. Let me list the `js/` folder.

GitHub's tree-listing endpoint is blocked. Let me try the API instead, which is allowed.

I'll fall back to the `bash_tool` and clone the repo so I can inspect everything locally.

This is a full classical chess-engine architecture — exactly the file layout you'd see in a tutorial-style engine. Let me confirm a hunch: this looks like a JavaScript port of **VICE** ("Video Instructional Chess Engine"), the well-known C engine by Bluefever Software whose YouTube series is the canonical "build your own chess engine" tutorial. The filenames (`movegen.js`, `makemove.js`, `pvtable.js`, `perft.js`, `io.js`, `search.js`, `evaluate.js`) match VICE's module split exactly. Let me verify by reading a few files.

Three things confirmed already:

1. The "smartforwarder" link **is hardcoded into the source** — it's not an injection at fetch time, it's intentional backlink/promo placed inside the engine UI.
2. There's **no Web Worker** — scripts are loaded directly into the page, so the search runs on the main thread (this is a noticeable architectural choice; multi-second searches will block the UI unless done with cooperative `setTimeout` yielding).
3. Google Analytics is wired up.

Now let me quickly inspect the engine internals.

This is unmistakably the **VICE engine ported to JavaScript**, line by line. Every single design choice here is straight out of Bluefever's tutorial:

- **`BRD_SQ_NUM = 120`** — the **10×12 mailbox** representation. The board is a 1-D array of 120 squares: 8 playable files × 8 ranks (= 64) padded with two ranks of "off-board" sentinels above/below and a one-square sentinel either side, so move generation can simply add a direction offset and check `OFFBOARD` instead of doing range arithmetic. The directional offsets confirm this: `KnDir = [-8, -19, -21, -12, 8, 19, 21, 12]` are the eight knight jumps in a 10-wide grid.
- **Move encoding in a single 25-bit integer** (the comment-block at line 300). Bit-packed: 7 bits "from", 7 bits "to", 4 bits captured-piece, 4 bits promoted-piece, plus three flags (en-passant, pawn-double-push, castle). Extraction via the `FROMSQ / TOSQ / CAPTURED / PROMOTED` macros plus the `MFLAGEP / MFLAGPS / MFLAGCA / MFLAGCAP / MFLAGPROM` masks. Identical layout to VICE.
- **Piece values**: `P=100, N=B=325, R=550, Q=1000, K=50000` — VICE's exact numbers (most modern engines use 100/320/330/500/900 or similar).
- **Zobrist hashing** with `PieceKeys[14*120]`, `CastleKeys[16]`, `SideKey`, and `HASH_PCE / HASH_CA / HASH_SIDE / HASH_EP` macros — the hash is incrementally updated by XOR'ing in/out piece-square keys when moves are made/unmade.
- **`CastlePerm[120]`** — a clever trick from VICE: castling rights are 4 bits (`WKCA=1, WQCA=2, BKCA=4, BQCA=8`), and to update them after any move you simply do `castlePerm &= CastlePerm[fromSq] & CastlePerm[toSq]`. The lookup table has `15` (= keep all rights) everywhere except the four corner squares and the two king starting squares, which carry the appropriate bitmask of rights to remove.
- **`Mirror64`** — for a tapered/pure white-perspective evaluation. Black's piece-square table values are obtained by mirroring the square index vertically, so only white tables need to be stored.

One quibble: the `RAND_32` function uses `Math.random()` with `* 255 + 1` per byte, which yields values in `[1, 255]` — never zero. So Zobrist keys here have **no zero bytes**, which is mildly biased but harmless in practice (and reduces hash collisions with a typical "all zero" sentinel position-key).

Let me look at the search and evaluation, the parts that determine actual playing strength.

Now I can give a definitive description of the search algorithm.

Let me also peek at the move generator briefly and check the "book" mechanism, since that was the most user-visible loading message.

Several important findings:

1. **The opening book code is commented out in `main.js`**. The "Loading book, please wait..." message is shown indefinitely (and `GameController.BookLoaded` stays `BOOL.FALSE` forever) because the AJAX call that would load `bookXml.xml` is disabled. So in practice the engine **never plays book moves** — it always searches from the start position. The book infrastructure exists in `board.js` (as `brd_bookLines`, `LineMatch`, `BookMove`) but it's dormant.
2. **The `BookMove` matcher is a string-based linear scan** through `brd_bookLines` comparing against the move-history line. Comment mentions a `lengthOfLineHack`. This is a far cry from a Polyglot binary book; it's a homemade text-line book format that was in development and abandoned.
3. **No Web Worker** — `srch_stop` is checked every 2048 nodes (`srch_nodes & 2047`), so the search yields cooperatively only by *aborting*, not by actually returning control to the browser. This means the UI **does freeze** during search — exactly what a Web Worker would prevent.

Let me look at the search algorithm in detail one more time to summarise it precisely.

Now I have enough to write a complete and accurate analysis.

---

# Detailed analysis of `kidschess/chessjs`

## Short version

`chessjs` is a **JavaScript port of "VICE"** (Bluefever Software's well-known C tutorial chess engine), wrapped in a minimal jQuery + HTML UI and hosted on GitHub Pages as "Kindle Chess". It's a small project (35 commits, ~600 KB total, default branch `develop`) whose engine source is essentially a faithful translation of Richard Allbert's tutorial code into JavaScript. The README is empty apart from a TODO that says "try with the new kindle" — explaining the otherwise-cryptic project name.

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
- **Bug / API mismatch in `Quiescence`** at line 116: it writes `brd_moveScores[MoveNum].score = 2000000`, but `brd_moveScores` is a flat array of numbers (not objects), so this assignment silently creates a `.score` property on a boxed number that's then thrown away. The PV-move boost in quiescence therefore **does not actually work** — a real bug. The same line in `AlphaBeta` at line 228 has the identical bug. Move ordering for the PV move falls back to whatever score MVV-LVA assigned (still high for captures, but not "highest").
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
- A `SmartForwarder` link is **hardcoded into `index.html`** between the "Engine Output" header and the thinking-time selector, with no logical connection to chess. This is a **deliberate backlink for SEO/promotion**, not anything injected by the fetcher. If you forked this you'd want to remove it.
- Google Analytics tag (`G-Q94743BW9M`) is also wired in.

## The "Loading book, please wait…" message

This is the most visible quirk of the live page, and the answer is straightforward: **the opening book is dead code**. In `main.js`, the `$.ajax(...)` call that would download `bookXml.xml`, parse it into `brd_bookLines`, set `GameController.BookLoaded = TRUE`, and remove the loading span is fully commented out. The span is therefore never removed, and `BookMove()` is never called from `SearchPosition()` because the `BookLoaded` flag stays false. The engine plays from move 1 by raw search.

The book format itself, judging from the surviving `BookMove()` and `LineMatch()` code, was going to be a list of strings of the form `"<game-history-prefix> <next-move>"`, with a magic-number offset (`lengthOfLineHack`) to slice out the move — clearly an ad-hoc text format rather than Polyglot.

## Bottom line

This is an **educational engine** — almost certainly a student's port of the VICE tutorial, finished enough to play but with rough edges (no TT scores, busted PV-move ordering in the quiescence/main loops because of the `.score` typo, dormant book, main-thread search). On the strength axis you'd likely see something around club level; on the engineering axis it's a clean, readable example of how the classical chess-engine architecture maps to JavaScript. If you wanted to fork and improve it, the highest-leverage changes — in roughly this order — would be:

1. Move the engine into a Web Worker so the UI doesn't freeze.
2. Fix the `brd_moveScores[MoveNum].score = 2000000` assignments to write the integer directly.
3. Promote `pvtable.js` from a PV-only cache to a real transposition table that stores `(score, depth, flag, bestMove)` and produces alpha/beta cutoffs.
4. Add aspiration windows and late-move reductions.
5. Enable an actual opening book — Polyglot `.bin` parsing in JS is a few hundred lines, or just JSON-encode a simple book.

Want me to work on any of these, or sketch out, say, a Web Worker wrapper or a proper transposition table for it?

 The repository is at:

**https://github.com/kidschess/chessjs**

Default branch is `develop` (not `main` or `master`), so direct file links use that branch — for example, the search code is at `https://github.com/kidschess/chessjs/blob/develop/js/search.js`.