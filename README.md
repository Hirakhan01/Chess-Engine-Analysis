# Chess Engine Analysis: Magnus Carlsen vs Hikaru Nakamura

An end-to-end data pipeline analyzing move-by-move precision of two elite chess players, built from scratch using the Lichess API and the Stockfish engine.

## Project Motivation

Built as a personal learning project to explore a new domain (chess) through data rather than gameplay. The goal was to quantify what "good" and "bad" chess actually looks like, using engine evaluation as ground truth, and to practice a full applied-analytics workflow — data collection, feature engineering, statistical analysis, and critically, catching and correcting my own errors along the way.

## Pipeline Overview

**1. Data Collection**
Pulled rated blitz games via the Lichess public API (`/api/games/user/{username}`), including move sequences, clock times, and player metadata. Sourced 50 games from Magnus Carlsen (`DrNykterstein`) and 16 from Hikaru Nakamura (`TSMFTXH`) — Hikaru's Lichess sample is smaller since his primary platform is chess.com.

**2. Board Reconstruction**
Used `python-chess` to replay each game's move sequence and reconstruct the board (FEN) after every move — necessary because Stockfish evaluates *positions*, not move lists.

**3. Engine Evaluation**
Connected to Stockfish via the UCI protocol, evaluating every position at depth 15. Computed centipawn loss per move — the gap between the move played and the engine's best available move.

**4. Move Classification**
Bucketed every move by centipawn loss into `good` / `inaccuracy` / `mistake` / `blunder`, using standard thresholds (0–49 / 50–99 / 100–299 / 300+).

**5. Batch Processing**
Processed 66 games (~2,959 moves total) with incremental CSV saving after each game, so a crash mid-run wouldn't lose completed work.

**6. Time-Pressure Analysis**
Aligned each move with clock time remaining to test whether error rate increases as players run low on time.

## Key Findings

- Across the full sample, Magnus and Hikaru's move quality was closely matched (77.8% vs 72.9% "good" moves; average centipawn loss 39.6 vs 49.4) — consistent with two of the strongest players in the world being separated by fine margins, not a large skill gap.
- Errors weren't randomly distributed across the board — they clustered on central/kingside squares (f3, f4, f5), where contested middlegame play concentrates.
- An initial "time pressure causes blunders" signal turned out to be a measurement artifact: extreme evaluation swings from mate-score encoding and already-decided positions (10+ pawns ahead/behind) were inflating the low-clock average by roughly 3x. After excluding already-decided positions, cp_loss stayed roughly flat across time buckets — no meaningful evidence of performance collapse under time pressure in this sample.

## Debugging & Engineering Notes

A few real problems worked through during the build, kept here because they were as instructive as the final results:
- **Sandboxed network restrictions** required moving execution to Google Colab, which introduced its own path-handling differences (Linux vs Windows Stockfish binaries).
- **Silent API failures**: an early version of the fetch function parsed a 404 error response as if it were valid game data, producing a phantom "1 game" result — fixed by adding explicit status-code checks before parsing.
- **Wrong username assumption**: assumed Hikaru's Lichess handle was `Hikaru`; the actual verified account (`TSMFTXH`) required confirming against Lichess's official records.
- **Mate-score distortion**: representing "mate in N" as a large numeric placeholder (100,000) caused a single move to register a ~900-pawn "blunder" and skew an entire game's average — required capping centipawn loss for aggregate statistics.
- **Stale variable/session bug**: a clock-time column silently failed to update after re-running earlier cells out of order, producing a flat, incorrect value across all rows until traced back and recomputed from scratch.

## Tech Stack

- **Python**: `python-chess`, `pandas`, `matplotlib`
- **Chess Engine**: Stockfish (via UCI protocol)
- **Data Source**: Lichess public API
- **Environment**: Google Colab

## Possible Extensions

- Larger sample size for Hikaru to enable a genuine time-pressure comparison between players
- Opening-repertoire analysis (which openings correlate with cleaner play)
- Extend classification thresholds to account for position complexity, not just raw centipawn loss

---

*Built as a self-directed learning project by Hirii — MSc Data Science & Analytics, University of Leeds.*
