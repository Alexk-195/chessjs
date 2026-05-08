# Kindle Chess

Read README.md for full project overview and setup instructions.

## File Reference

| File | Purpose |
|------|---------|
| `index.html` | Main entry point; loads all scripts and renders the board |
| `stylesheets/styles.css` | Board and piece styling |
| `js/defs.js` | Constants and enumerations (pieces, colours, squares, move flags) |
| `js/board.js` | Board state variables and initialisation |
| `js/makemove.js` | Apply/undo moves on the board |
| `js/movegen.js` | Legal move generation |
| `js/evaluate.js` | Static position evaluation |
| `js/pvtable.js` | Principal variation (PV) table for move ordering |
| `js/search.js` | Alpha-beta search |
| `js/perft.js` | Perft move-count tests for engine verification |
| `js/gui.js` | DOM manipulation and board rendering |
| `js/io.js` | FEN parsing and move-string conversion |
| `js/protocol.js` | UCI-style protocol handling |
| `js/client.js` | Browser-side glue between GUI and engine |
| `js/main.js` | Startup and event wiring |
| `js/jquery-1.10.1.min.js` | Bundled jQuery (no CDN dependency) |
| `images/` | PNG piece sprites (wP, wN, wB, wR, wQ, wK, bP…) |
| `dev.sh` | Local dev helper script |
| `analysis.md` | Notes and analysis scratch-pad |
