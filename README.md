# tiny-terminal

A tiny fake terminal, built to learn how Ghostty's workers pass notes
(mailboxes + doorbells) and share one screen safely (a lock).

You build it yourself, stage by stage. Each stage teaches one idea.
Make a stage work before moving to the next.

```
npm start           # run src/main.ts
npm run dev         # re-run on every save
npm run typecheck   # let TypeScript check your work
```

---

## The cast

| Worker | Ghostty name | Job |
|---|---|---|
| **Producer** | the program (your shell) | Spits out lines fast, e.g. `[RED]error: disk full` |
| **Parser** | termio + terminal (the chef) | Strips color tags, keeps a "screen": the last 20 lines + colors |
| **Renderer** | renderer (the photographer) | Draws the screen to your console |
| **Keyboard** | apprt / Surface (the window) | Turns keypresses into notes for the other workers |

---

## Part 1: the tiny terminal

### Stage 1: one plain loop
Producer, parser and renderer all run in one loop. Redraw after every line.

- [ ] It works, and you can see the last 20 lines with colors
- [ ] **Feel it:** it's slow and flickery. That's the problem you're solving.

### Stage 2: mailbox + doorbell
Write your own `Mailbox` and `Doorbell`. The parser rings the renderer's
doorbell after each line. The renderer sleeps until woken.

- [ ] A doorbell rung many times while the worker is busy wakes it **once**
- [ ] **Check:** count rings and draws. 10,000 lines should give far fewer
      than 10,000 draws. Equal counts mean your merging doesn't work.

### Stage 3: notes with kinds
Keyboard input becomes notes: `{ kind: "pause" }`, `{ kind: "clear" }`,
`{ kind: "speed", value: 5 }`. Use a discriminated union and a `switch`.

- [ ] **Check:** add a new note kind and see TypeScript complain until you
      handle it everywhere.

### Stage 4: real threads
Move the producer and the parser into `worker_threads`. Connect them with
`postMessage`.

- [ ] **Check:** typing stays responsive while the producer floods output.
      If typing lags, something is blocking.

### Stage 5: a mailbox with a size limit
Give the parser's tray a maximum (say 64). When it's full: does the producer
wait, or are notes dropped? Try both.

- [ ] You can explain why Ghostty's mailbox is fixed-size
- [ ] You can explain the "put the lock down while waiting" edge case
      (see `src/termio/mailbox.zig` in Ghostty)

### Stage 6 (stretch): share the screen with a lock
Put the screen in a `SharedArrayBuffer` and build a lock with `Atomics`.
The parser writes under the lock. The renderer copies quickly under the
lock, then draws without it.

- [ ] **Check:** add a fake slow draw (50ms) **inside** the lock. Then move it
      **outside**. Measure how much faster the parser gets.

### Measure at the end of every run

- lines produced
- notes sent
- doorbell rings
- actual draws
- total time
- longest wait for the lock (stage 6)

---

## Part 2: make it fun (Snake)

### Snake inside your terminal
- The **game** is the producer: every tick it outputs the board as text with
  color tags (`[GREEN]████`, `[RED]●`).
- **Arrow keys** become notes (`{ kind: "key", dir: "up" }`) that reach the game.

### Lag meter
A status line at the bottom:

```
key→screen: 12ms   draws/s: 58   rings/s: 3400   dropped notes: 0
```

`key→screen` is the important one: time from keypress to seeing the snake turn.

### Sabotage switches
Number keys that break one thing at a time, mid-game:

| Key | Sabotage |
|---|---|
| `1` | Hold the lock during the slow draw |
| `2` | Turn off doorbell merging (draw on every ring) |
| `3` | Shrink the key mailbox to size 1 |
| `4` | Parser randomly freezes for 200ms |
| `5` | Draw straight from the live screen (no snapshot) |
| `0` | Fix everything |

### What you should feel

| If this is broken... | ...you feel this |
|---|---|
| Keys wait behind a busy worker | Snake turns too late and crashes |
| Rings don't merge | Flicker and stutter |
| Lock held while drawing | Game hitches and slows down |
| Mailbox too small, notes dropped | Quick ↑ then → only turns once |
| No snapshot | Torn frames: half old board, half new |

### Later ideas
- **Matrix rain**: pure stress test for the renderer
- **Typing race**: any input lag ruins it
- **Two snakes**: arrows + WASD, two senders sharing one mailbox

---

## Node gotchas (so you don't lose an evening)

- **Node runs `.ts` directly** (Node 24+), but only "erasable" TypeScript.
  No `enum`, no `namespace`, no constructor parameter properties
  (`constructor(private x: number)`). `npm run typecheck` will catch these.
- **Imports need the `.ts` extension:** `import { Mailbox } from "./mailbox.ts";`
- **Type-only imports** need `import type { ... }`.
- **Workers (stage 4):**
  `new Worker(new URL("./parser.ts", import.meta.url))`
- **Arrow keys:** `process.stdin.setRawMode(true)`, then listen to `data`.
  Arrow keys arrive as escape sequences (`\x1b[A` = up). That's the same
  kind of escape sequence Ghostty's parser handles! Raw mode needs a real
  terminal, so run it in a terminal window, not an IDE output pane.
  Remember to handle `Ctrl+C` (`\x03`) yourself in raw mode.
- **Redrawing the console:** `\x1b[2J` clears the screen, `\x1b[H` moves
  the cursor to the top-left. `\x1b[31m` is red, `\x1b[0m` resets.
- **Shared memory (stage 6):** `Atomics.wait` works in Node and in workers.

---

## Where this comes from in Ghostty

| Idea | Ghostty file |
|---|---|
| Mailbox + doorbell | `src/termio/mailbox.zig`, `src/datastruct/blocking_queue.zig` |
| Note kinds | `src/termio/message.zig`, `src/renderer/message.zig` |
| Doorbell with no note | `queueRender` in `src/termio/stream_handler.zig` |
| Quick snapshot under lock | `updateFrame` in `src/renderer/generic.zig` |
| Two-stage reader | `ReadThread` in `src/termio/Exec.zig` |
