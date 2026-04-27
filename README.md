# How I Built Three Classic Games with Real AI Brains — A Deep Dive into the Algorithms

> *From Minimax to Probabilistic Constraint Solving — the engineering behind a Node.js Game Hub*

---

When most developers say they "added AI" to a game, they mean the enemy moves randomly. I wanted to do something different. In my **Game Hub** project — a Node.js + Express server hosting Tic-Tac-Toe, Minesweeper, and Battleship — I implemented a distinct AI algorithm for each game, each one carefully chosen to match the nature of the problem. This post is a technical deep-dive into those algorithms: how they work, why I picked them, and what the code actually looks like under the hood.

---

## The Project at a Glance

Game Hub is a full-stack web application built on **Node.js and Express**, serving three browser-based games from a single server. The frontend is plain HTML, CSS, and vanilla JavaScript. All game logic — including every AI decision — lives on the server, keeping the client lightweight and the rules tamper-proof.

Each game has its own Express router under `server/games/`, and each one solves a fundamentally different AI problem:

| Game | AI Problem Type | Algorithm Used |
|---|---|---|
| Tic-Tac-Toe | Two-player zero-sum game | **Minimax** |
| Minesweeper | Constraint satisfaction + uncertainty | **CSP Solver + Probabilistic Risk Estimation** |
| Battleship | Partial-information search | **Parity Hunt-and-Target with Pending-Hit Tracking** |

Let's go deep on each one.

---

## Game 1: Tic-Tac-Toe — The Minimax Algorithm

### What Problem Are We Solving?

Tic-Tac-Toe is a **two-player, zero-sum, perfect-information game**. Both players know the full board state at all times. There is no luck, no hidden information. This makes it the canonical textbook case for the **Minimax algorithm**.

### How Minimax Works

Minimax is a recursive decision-making algorithm that models the game as a tree. Every node in the tree is a board state. Every edge is a move. The AI works by imagining all possible futures — every move it could make, every counter-move the human could make in response, and so on — all the way to the end of the game.

At each leaf node (a finished game), the algorithm assigns a score:
- **+10** if the AI wins
- **−10** if the human wins
- **0** for a draw

The score is also offset by **depth**, meaning the AI prefers to win faster and lose slower (a win in 2 moves scores `10 - 2 = 8`; a win in 4 moves scores `10 - 4 = 6`). This elegantly prevents the AI from making pointless delaying moves.

The algorithm then propagates scores back up the tree using two alternating rules:
- When it is the **AI's turn** (the *maximizing* player), it picks the child node with the **highest** score.
- When it is the **human's turn** (the *minimizing* player), it picks the child node with the **lowest** score.

This is exactly what a rational opponent would do — always play their best move — so Minimax finds the optimal strategy against a perfect opponent.

### The Code

```javascript
function minimax(board, maximizing, depth = 0) {
  const winner = winnerOf(board);
  if (winner === "O") return { score: 10 - depth, move: null };
  if (winner === "X") return { score: depth - 10, move: null };
  if (isDraw(board)) return { score: 0, move: null };

  let bestMove = null;
  if (maximizing) {
    let bestScore = -Infinity;
    for (let i = 0; i < 9; i += 1) {
      if (board[i] !== "") continue;
      board[i] = "O";
      const result = minimax(board, false, depth + 1);
      board[i] = "";
      if (result.score > bestScore) {
        bestScore = result.score;
        bestMove = i;
      }
    }
    return { score: bestScore, move: bestMove };
  }

  let bestScore = Infinity;
  for (let i = 0; i < 9; i += 1) {
    if (board[i] !== "") continue;
    board[i] = "X";
    const result = minimax(board, true, depth + 1);
    board[i] = "";
    if (result.score < bestScore) {
      bestScore = result.score;
      bestMove = i;
    }
  }
  return { score: bestScore, move: bestMove };
}
```

Notice that the board is mutated in place and then immediately restored (`board[i] = "O"` → recurse → `board[i] = ""`). This is a classic backtracking pattern. No new board objects are allocated on every recursive call, keeping memory usage minimal.

### Why Minimax Is Perfect for Tic-Tac-Toe

The game tree for Tic-Tac-Toe has at most **9! = 362,880 leaf nodes**, which is trivially small. The full tree can be explored in microseconds. For larger games like Chess, you would need enhancements like **Alpha-Beta Pruning** (which discards branches that cannot possibly improve the current best result) to make Minimax tractable. Here, the brute-force approach is both correct and fast enough that no pruning is needed.

The result: the AI **never loses**. It always finds the optimal move. If you play perfectly, you get a draw. If you make any mistake, you lose.

---

## Game 2: Minesweeper — Constraint Satisfaction + Probabilistic Risk Estimation

### What Problem Are We Solving?

Minesweeper is a fundamentally different kind of AI problem. It is a **single-player game of incomplete information**. The board contains hidden mines, and the AI (acting as a helper or auto-player) must reason about what it cannot see. This makes it a **Constraint Satisfaction Problem (CSP)** combined with **probabilistic reasoning** when no deterministic move exists.

### Phase 1: Deterministic CSP Solving

The board contains numbered cells. Each number tells you exactly how many mines are among its (up to 8) neighbors. The AI builds a set of **constraints** from this information.

For every revealed numbered cell, the AI computes:
1. Which of its neighbors are unrevealed (unknown)?
2. How many of those must be mines (the clue minus already-flagged neighbors)?

This yields constraints of the form: *"among cells {A, B, C}, exactly 2 are mines."*

Two deterministic rules immediately follow:

**Rule 1 — Safe Cell:** If `minesLeft == 0` for a constraint, all unknown cells in that group are definitively safe. Reveal them immediately.

**Rule 2 — Mine Cell:** If `minesLeft == unknownCount` for a constraint, every unknown cell in the group is definitively a mine. Flag them all.

```javascript
function buildConstraints(game) {
  const safe = new Set();
  const mines = new Set();
  const constraints = [];

  for (let row = 0; row < game.size; row++) {
    for (let col = 0; col < game.size; col++) {
      if (!game.revealed[row][col]) continue;
      const clue = game.adj[row][col];
      if (clue <= 0) continue;

      const unknown = [];
      let flaggedCount = 0;
      for (const [nr, nc] of neighbors(game.size, row, col)) {
        if (game.flagged[nr][nc]) flaggedCount++;
        else if (!game.revealed[nr][nc]) unknown.push(keyOf(nr, nc));
      }

      const minesLeft = clue - flaggedCount;
      if (minesLeft === 0) {
        for (const cell of unknown) safe.add(cell);
      } else if (minesLeft === unknown.length) {
        for (const cell of unknown) mines.add(cell);
      } else if (minesLeft > 0 && minesLeft < unknown.length) {
        constraints.push({ cells: unknown, minesLeft });
      }
    }
  }

  return { safe, mines, constraints };
}
```

The AI always exhausts all deterministic moves first. Only when no safe or mine cell can be proven does it move to the probabilistic phase.

### Phase 2: Probabilistic Risk Estimation

When no deterministic move exists, the AI must guess — but it guesses intelligently. It estimates a **mine probability** for every unrevealed, unflagged cell and picks the one with the lowest estimated risk.

The probability estimate is computed from the active constraints:

- **Local constraint risk:** For each constraint `{ cells, minesLeft }`, the naive local probability for any cell in that group is `minesLeft / cells.length`.
- **Global base risk:** For cells not covered by any constraint, the risk is `remainingMines / totalHiddenCells`.
- **Blended risk:** For constrained cells, the AI uses a blended formula that incorporates both local and global signals, ensuring even constrained cells are not rated below 60% of the global baseline.

```javascript
function chooseBestRiskMove(game, constraints) {
  const riskByCell = new Map();
  const countByCell = new Map();

  const remainingMines = Math.max(0, game.mineCount - countFlags(game));
  const defaultRisk = hidden.length > 0 ? remainingMines / hidden.length : 1;

  for (const constraint of constraints) {
    const localRisk = constraint.minesLeft / constraint.cells.length;
    for (const cell of constraint.cells) {
      riskByCell.set(cell, (riskByCell.get(cell) || 0) + localRisk);
      countByCell.set(cell, (countByCell.get(cell) || 0) + 1);
    }
  }

  let best = null;
  for (const [row, col] of hidden) {
    const key = keyOf(row, col);
    const localCount = countByCell.get(key) || 0;
    const avgLocalRisk = localCount > 0 ? riskByCell.get(key) / localCount : defaultRisk;
    const risk = localCount > 0
      ? Math.max(avgLocalRisk, defaultRisk * 0.6)
      : defaultRisk;

    if (!best || risk < best.risk) {
      best = { row, col, risk, coverage: localCount };
    }
  }
  return best;
}
```

### Opening Policy

There is one more heuristic: on the very first move, before anything is revealed, the AI reveals the **center cell**. This is an information-maximization heuristic. The center cell has 8 neighbors and is adjacent to the largest fraction of the board, so revealing it maximizes the expected number of cells that get cascade-revealed via flood-fill.

### Flood-Fill Reveal (BFS/DFS)

When an empty cell (adjacency = 0) is revealed, all neighboring cells should also be revealed recursively, and if any of those are also empty, their neighbors should be revealed too. This is implemented as an **iterative depth-first search using a stack** to avoid call-stack overflow on large open areas:

```javascript
function floodReveal(game, startRow, startCol) {
  const stack = [[startRow, startCol]];
  while (stack.length > 0) {
    const [row, col] = stack.pop();
    if (!inBounds(game.size, row, col)) continue;
    if (game.revealed[row][col] || game.flagged[row][col]) continue;
    if (game.mines[row][col]) continue;

    game.revealed[row][col] = true;
    game.revealedSafeCells++;

    if (game.adj[row][col] !== 0) continue; // Only expand from empty cells
    for (const [nr, nc] of neighbors(game.size, row, col)) {
      if (!game.revealed[nr][nc] && !game.flagged[nr][nc]) {
        stack.push([nr, nc]);
      }
    }
  }
}
```

The combined result is a Minesweeper AI that plays essentially like an expert human: it never makes an avoidable mistake and only gambles when forced to by the mathematics of the board.

---

## Game 3: Battleship — Parity Hunt-and-Target with Pending-Hit Tracking

### What Problem Are We Solving?

Battleship is a game of **sequential decision-making under partial information**. The AI (the computer opponent) must find and sink five ships hidden on a 10×10 grid by firing shots one at a time. Each shot reveals only whether it was a hit or a miss — no other information. This is a problem in **information-theoretic search** and **heuristic state machines**.

The AI operates in two distinct modes that switch based on what it has learned.

### Mode 1: Hunt Mode — Parity-Based Search

When the AI has no active hits to follow up, it is in "Hunt Mode." Its goal is to find any ship as efficiently as possible.

The key insight: **every ship has a length of at least 2**. This means no ship can occupy only odd-indexed cells (where row + col is odd) and only even-indexed cells (where row + col is even) simultaneously. If you shoot only cells where `(row + col) % 2 === 0` — a checkerboard pattern — you are guaranteed to hit every ship of size 2 or larger.

This **parity trick** cuts the search space in half. On a 10×10 grid, instead of needing up to 100 shots to find all ships, you need at most 50 shots just from the parity pass to guarantee touching every ship at least once.

The AI precomputes two shuffled pools at game creation:

```javascript
function shuffledCoordinates() {
  const parity = [];
  const fallback = [];
  for (let row = 0; row < SIZE; row++) {
    for (let col = 0; col < SIZE; col++) {
      if ((row + col) % 2 === 0) parity.push([row, col]);
      else fallback.push([row, col]);
    }
  }
  // Fisher-Yates shuffle both pools
  for (let i = parity.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [parity[i], parity[j]] = [parity[j], parity[i]];
  }
  // ... same for fallback
  return { parity, fallback };
}
```

The AI exhausts the parity pool first, then falls back to the odd-parity cells only if needed. Each pool is shuffled using the **Fisher-Yates algorithm** (also known as Knuth Shuffle), guaranteeing a uniform random ordering with O(n) time complexity and no bias.

### Mode 2: Target Mode — Pending-Hit Tracking

When the AI scores a hit, it switches to "Target Mode." Now it must determine the ship's orientation and sink it efficiently.

The AI maintains a `pendingHits` set — the set of coordinates that were hit but have not yet been attributed to a sunken ship. This is the core of Target Mode logic.

```javascript
function targetFromPendingHits(game) {
  const pending = [...game.aiKnowledge.pendingHits].map(parseKey);
  if (pending.length === 0) return null;

  const byRow = new Map();
  const byCol = new Map();
  for (const [row, col] of pending) {
    if (!byRow.has(row)) byRow.set(row, []);
    if (!byCol.has(col)) byCol.set(col, []);
    byRow.get(row).push(col);
    byCol.get(col).push(row);
  }

  const candidates = new Map();

  // If 2+ hits are aligned in a row, extend the line at both ends (score: 10)
  for (const [row, cols] of byRow) {
    if (cols.length < 2) continue;
    const sorted = cols.sort((a, b) => a - b);
    addCandidate(candidates, game, row, sorted[0] - 1, 10);
    addCandidate(candidates, game, row, sorted[sorted.length - 1] + 1, 10);
  }
  // Same logic for column alignment
  for (const [col, rows] of byCol) { /* ... */ }

  // Fall back: shoot adjacent to any known hit (score: 5)
  for (const [row, col] of pending) {
    addCandidate(candidates, game, row - 1, col, 5);
    addCandidate(candidates, game, row + 1, col, 5);
    addCandidate(candidates, game, row, col - 1, 5);
    addCandidate(candidates, game, row, col + 1, 5);
  }

  const ordered = [...candidates.values()].sort((a, b) => b.score - a.score);
  return [ordered[0].row, ordered[0].col];
}
```

The logic has a clear priority hierarchy:
1. **Extend a confirmed line** (two or more collinear hits): Score 10. If we have two hits in a row, the ship is horizontal — keep shooting along that axis.
2. **Probe adjacent to a lone hit**: Score 5. If we only have one hit, shoot orthogonally until we find the ship's direction.

When a ship is **sunk**, all of its cells are removed from `pendingHits`, cleanly separating that ship from any remaining unsunk hits (which would belong to a different ship).

### State Machine Summary

```
[Game Start]
     │
     ▼
[HUNT MODE: Shoot from parity pool]
     │  Hit?
     ▼
[TARGET MODE: Extend line or probe neighbors]
     │  Ship sunk? Clear pendingHits
     ▼
[Back to HUNT if pendingHits empty, else continue targeting]
```

This two-mode state machine, combined with the parity optimization, produces an AI that plays Battleship at near-expert level — significantly better than a random shooter.

---

## The Voice Layer: Playing with Your Words

Beyond the AI algorithms, every game in the hub supports **voice-controlled gameplay** using the Web Speech API — specifically `SpeechRecognition` for input and `SpeechSynthesis` for spoken feedback. This transforms Game Hub from a click-only experience into a genuinely hands-free one.

### How It Works: The Web Speech API

All three games share the same foundational setup:

```javascript
const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
const recognition = SpeechRecognition ? new SpeechRecognition() : null;

if (recognition) {
  recognition.lang = "en-US";
  recognition.interimResults = false;
  recognition.continuous = false;
}
```

The `continuous: false` setting means the microphone activates only on demand — the player clicks the **Voice** button, says their command, and recognition stops. This avoids accidental triggers mid-game.

For spoken feedback, a shared `speak()` utility wraps the `SpeechSynthesis` API:

```javascript
function speak(message) {
  if (!("speechSynthesis" in window)) return;
  window.speechSynthesis.cancel();
  window.speechSynthesis.speak(new SpeechSynthesisUtterance(message));
}
```

Calling `cancel()` before each utterance ensures that rapid state changes (like an instant AI counter-move) don't queue up stale announcements — only the most current message gets spoken aloud.

### Tic-Tac-Toe: "Play 5"

Tic-Tac-Toe uses the simplest voice grammar. The board cells are numbered 1–9 left-to-right, top-to-bottom, so the only command needed is:

> **"Play [number]"** — e.g., *"play 5"* to take the center square.

The parser uses a focused regex to extract the move number:

```javascript
function parsePlayCommand(command) {
  const match = command.toLowerCase().match(/\bplay\s+(\d+)\b/);
  if (!match) return null;
  return Number(match[1]);
}
```

If the player says a cell that is already occupied, the game speaks: *"play other move, move already played"* — giving immediate, clear feedback without requiring the player to look at the screen. Cells on the board also display their numbers while empty, so players can quickly map speech to position without memorizing a layout.

### Minesweeper: Two-Action Voice Commands

Minesweeper is more complex because each cell can be either **revealed** or **flagged** — two fundamentally different actions. The voice grammar reflects this:

> **"Reveal [number]"** / **"Open [number]"** / **"Move [number]"** — uncovers the cell.
> **"Flag [number]"** — toggles a mine flag on the cell.

The parser handles all these synonyms in a single regex:

```javascript
function parseVoiceCommand(command) {
  const match = command.toLowerCase().match(/\b(flag|reveal|open|move)\s+(\d+)\b/);
  if (!match) return null;
  return {
    action: match[1] === "open" || match[1] === "move" ? "reveal" : match[1],
    moveNumber: Number(match[2]),
  };
}
```

Mapping `"open"` and `"move"` to `"reveal"` is a deliberate UX decision — people naturally say "open cell 12" or "move to 14," so the parser meets them where they are rather than enforcing a rigid vocabulary.

The voice feedback is context-aware. If a player tries to reveal a flagged cell, the game speaks: *"block 12 is flagged. say flag 12 to unflag first."* This prevents accidental reveals of intentionally marked mines without making the player look up from the board. The toggle behavior for flagging is also announced: cells that are already flagged will be unflagged when the flag command is repeated — and the spoken status confirms which action just happened.

### Battleship: Grid Navigation by Number

Battleship operates on a 10×10 grid, meaning cells are numbered 1–100 (row by row). The voice command mirrors Tic-Tac-Toe's simple grammar:

> **"Play [number]"** — fires at that cell on the enemy grid.

The parser is identical to Tic-Tac-Toe's `parsePlayCommand`. The conversion from cell number to grid coordinates is handled by:

```javascript
function cellNumberToPosition(cellNumber) {
  const index = cellNumber - 1;
  return {
    row: Math.floor(index / state.boardSize),
    col: index % state.boardSize,
  };
}
```

This means *"play 45"* maps to row 4, column 4 on the 10×10 enemy board — no grid-letter notation required. Players don't need to say "B-7" or any alpha-numeric coordinate — just a number that corresponds to the visible cell label.

### Graceful Degradation

Not all browsers support the Web Speech API (notably Firefox desktop). The code handles this gracefully: if `SpeechRecognition` is unavailable, the voice button is **disabled** and a status message informs the player that speech recognition isn't supported. No errors are thrown, and the game remains fully playable via clicks.

```javascript
} else {
  voiceBtn.disabled = true;
  setVoiceStatus("speech recognition is not supported in this browser.");
}
```

This is a key principle of progressive enhancement: voice is a layer of convenience on top of a fully functional core, not a dependency of it.

### Why Voice Input Matters Here

Adding voice to these games wasn't just a novelty feature. It served a concrete design goal: keeping the experience accessible and immersive. For Minesweeper in particular — where you're constantly scanning the board and flagging suspected mines — being able to say *"flag 34"* without moving your mouse reduces cognitive load. For Battleship, calling out shot coordinates like a naval officer genuinely adds to the atmosphere.

The Web Speech API made this feasible in pure vanilla JavaScript, with no external libraries and no backend involvement. All recognition happens in the browser, and the spoken feedback is instantaneous. It is a surprisingly powerful API for how little code it takes to integrate.

---

## Putting It All Together: AI by Design

What makes this project interesting from an AI perspective is that each game demanded a completely different algorithmic approach:

- **Tic-Tac-Toe** is solved by **exhaustive game-tree search** (Minimax). The problem is small enough that brute force is optimal.
- **Minesweeper** is solved by **logical inference first, probabilistic estimation second**. The AI deduces what it can, then makes the mathematically safest guess when deduction fails.
- **Battleship** is solved by **information-theoretic search heuristics** — parity-based coverage in Hunt mode and pattern-directed targeting in Target mode.

None of these algorithms use machine learning. They are all classical AI — hand-crafted reasoning systems that encode domain knowledge about each game into precise, deterministic (or probabilistically principled) decision procedures. This is sometimes called "Good Old-Fashioned AI" (GOFAI), and for well-defined games with small state spaces, it remains the right tool for the job.

The voice layer sits on top of all of this as a pure UX enhancement — bridging human natural language to the same backend API calls that click-based input uses. The architecture stays clean because voice input is just another event source feeding the same game logic.

---

## What's Next

There are several natural extensions to explore:

- **Alpha-Beta Pruning** for Minimax — if the board were larger (4×4, 5×5), pruning would become essential.
- **Subset constraint propagation** for Minesweeper — when one constraint's cell set is a subset of another's, you can derive additional deterministic inferences that the current solver misses.
- **Probability density maps** for Battleship Hunt mode — instead of a fixed parity strategy, model the full probability distribution of where ships could be placed and always shoot the highest-probability cell.
- **Continuous voice mode** — enabling `recognition.continuous = true` with wake-word detection would allow fully hands-free play session without needing to press the voice button between every move.
- **Richer voice feedback** — reading out board state on demand (e.g., *"what's around cell 34?"* in Minesweeper) could make the game playable with eyes closed.

The full source code for this project is available on GitHub. If you found this breakdown useful, give it a follow — more deep-dives on classical AI and game theory are coming.

---

*Tags: #AI #GameDevelopment #Algorithms #NodeJS #JavaScript #Minimax #ConstraintSatisfaction #WebSpeechAPI #VoiceControl #Programming #SoftwareDevelopment*
