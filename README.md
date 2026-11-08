# Echo

A meeting workspace where the conversation builds the canvas. People join a room, talk on video, and share one
visual canvas. What they say becomes ideas, questions, decisions, tasks, problems and topics on the canvas, linked
to each other as the discussion develops. They can also tell Echo what to change in plain language, and edit by hand.

The whole project is one file, `echo.js`, with no npm dependencies.

## What it does

- **Video and audio in the room.** Peer-to-peer calls (WebRTC) with camera and microphone toggles.
- **The conversation builds the canvas.** Each spoken or typed sentence is classified and placed:
  - topics ("Let's talk about pricing")
  - problems ("The problem is that plans are confusing")
  - ideas ("What if we offered a free tier?")
  - questions ("How should we price the team plan?")
  - decisions ("We decided to go with three tiers"), linked to the question they answer
  - tasks with owner and due date ("I'll draft the page by Friday", "Priya will update the calculator by next week")
  - Repeating a point raises a counter on the existing item instead of adding a duplicate.
  - Filler and chit-chat ("yeah okay", "can you hear me?") add nothing.
- **Tell Echo what to change.** Start with "echo" (or `/`), by voice or in the chat box:

  | Say | Result |
  |---|---|
  | `echo, group these ideas` | Groups the selected items. Without a selection: `group the ideas`, `group the pricing ideas`, `group everything by type` |
  | `echo, move the product tour under the problem` | Re-parents an item (loops are refused) |
  | `echo, visualize this` | Lays the selection (or the latest items, or a node's children) out as a flow with "then" links |
  | `echo, connect A to B` | Adds a link |
  | `echo, rename X to Y`, `mark that as a decision`, `delete the last idea` | Edits |
  | `echo, summarize`, `list the action items`, `what are the open questions` | Adds a summary note to the canvas |
  | `echo, undo` | Reverts the last change (also the Undo button and Ctrl/Cmd+Z) |
  | `echo, help` | Lists commands |

  Items are referred to by selection ("this", "these"), by type ("the problem", "the last idea"), or by keywords
  ("the welcome email"). Anything not understood gets a reply that says so.
- **Real-time collaboration.** Shared cursors with names, other people's selections outlined in their colour, live
  dragging, a shared undo stack, and **Follow Me**: one person presents and everyone follows their view; panning or
  zooming stops following, and the Follow button re-syncs.
- **Persistent rooms.** The canvas and conversation are saved to disk. Leave, come back later with the same room
  name, and everything is there.
- **Export.** `GET /api/rooms/<room>/summary.md` (or `.json`) returns decisions, action items with owners, open
  questions, problems and ideas.

## Run it

Requirements: Node.js 18 or newer (Node 22 to run the tests).

```bash
node echo.js
# open http://127.0.0.1:3000 in two browser windows, join the same room name
```

Browsers allow camera and microphone only on `localhost` or HTTPS. To use Echo across machines, put a reverse proxy
with HTTPS in front of it and set `HOST=0.0.0.0` (or run the proxy on the same machine). Echo itself does not do TLS.

For talking to the canvas, use **Chrome or Edge** and press **Transcribe**; other browsers can use the chat box.

## Configuration

Environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `3000` | Port |
| `HOST` | `127.0.0.1` | Address to listen on |
| `DATA_DIR` | `./data` | Where rooms are saved (`room-<name>.json`) |
| `ALLOWED_ORIGINS` | none | Extra origins allowed to open a WebSocket (same-origin is always allowed) |
| `MAX_CLIENTS_PER_ROOM` | `8` | Video is a peer-to-peer mesh; keep this small |
| `STUN_URLS` | `stun:stun.l.google.com:19302` | Comma-separated STUN servers |
| `TURN_URL`, `TURN_USERNAME`, `TURN_CREDENTIAL` | none | TURN relay, needed on some networks |
| `INTERPRETER_URL`, `INTERPRETER_API_KEY`, `INTERPRETER_MODEL`, `INTERPRETER_TIMEOUT_MS` | none, 8000 | Optional remote interpreter (see below) |
| `ECHO_QUIET` | unset | `1` silences the JSON logs |

## Remote interpreter (optional)

By default Echo understands conversation with built-in rules (English). If `INTERPRETER_URL` points to an endpoint
that speaks the chat-completions JSON format, Echo sends it each utterance together with a compact view of the
canvas and applies the edits it returns. This helps with phrasing the rules do not know.

- The reply must be a JSON object of canvas edits. Every edit goes through the same validation as a person's edit
  (known operations only, existing ids, size limits, no loops), so a bad or hostile reply can at worst add junk that
  Undo removes.
- Meeting speech is untrusted input. The request tells the interpreter to ignore instructions inside it.
- If the endpoint is down, slow, or returns something unusable, Echo falls back to the built-in rules.
- Commands that the rules already understand never leave your server.
- **This path was tested with a stand-in endpoint only, never against a live service.** Try it with your own
  endpoint before relying on it. Utterances and canvas text are sent to whatever endpoint you configure.

## How it is built

`echo.js`, top to bottom: configuration; canvas model and operations; layout; summaries; the language interpreter and
commands; a WebSocket (RFC 6455) implementation; rooms and persistence; the HTTP server; and the browser client
(served at `/`).

- **Canvas.** Nodes, parent/child edges (a tree), link edges, and groups. All changes are validated operations
  (`addNode`, `moveNode`, `setParent`, `addGroup`, ...). Limits: 2000 nodes, 6000 edges, 100 groups, 500 characters
  per item.
- **Layout.** Items nobody has dragged are placed automatically: trees grow left to right, parents centre on their
  children, groups form blocks to the right, flows lay out in a row. Dragging an item pins it.
- **Protocol.** JSON over one WebSocket at `/ws`. Client messages: `join`, `cursor`, `select`, `ops`, `say`, `undo`,
  `present`, `view`, `follow`, `media`, `signal`, `ping`. Server messages: `welcome`, `peer.join`, `peer.leave`,
  `peer.update`, `cursor`, `select`, `canvas` (full state after a change), `move` (during drags), `transcript`,
  `presenter`, `view`, `signal`, `notice`, `error`, `pong`.
- **Video.** The page connects every pair of participants directly; Echo only relays the setup messages. Camera and
  microphone can be switched on later without renegotiating.
- **Ordering.** Conversation in a room is processed one utterance at a time, in the order spoken.
- **Persistence.** Saved shortly after each change, when a room empties, and on shutdown, by writing a temporary file
  and renaming it, so a crash never leaves a half-written room. A damaged room file is ignored and the room starts empty.

## Security and privacy

- There are no accounts. **Anyone who knows a room name can join it**, so use hard-to-guess room names for private
  discussions. Rooms hold up to `MAX_CLIENTS_PER_ROOM` people.
- WebSocket connections from other websites are refused (origin check). The page is served with a content security
  policy; text from people is always inserted as plain text.
- Messages are size-limited (256 KB), rate-limited per connection, and malformed messages are ignored or close the
  connection with the right WebSocket code. A participant who cannot keep up is disconnected.
- **Speech recognition is done by the browser.** In Chrome and Edge it sends audio to the vendor's cloud service.
  Typing in the chat box does not.
- Video is peer to peer, so participants can see each other's network addresses. The default STUN server is a public
  one; set `STUN_URLS` to your own if that matters.
- Not included: user accounts, per-room passwords, TLS, rate limits across connections, multiple server processes
  (rooms live in one process).

## Tests

```bash
node echo.test.js
```

49 tests, using Node's built-in test runner. They cover:

- canvas operations, limits, loops, and recovery from malformed saved data
- layout
- sentence classification, owners and due dates, conversation-to-canvas linking, duplicate handling
- every command
- the WebSocket implementation, including fragmentation, length encodings, and every protocol violation
- end to end with real sockets: joining, presence, cursors, speaking to build the canvas, drags and undo, Follow Me,
  video signalling, restart persistence, damaged files, oversized and malformed messages, floods, HTTP routes, the
  origin check, and the remote interpreter with fallback
- the browser page's script, run against the real server in a stand-in DOM

**Not covered by the tests:** how the page looks (layout, colours, responsiveness); real browsers; real video and
audio calls; real speech recognition; a live remote interpreter; many people at once. Open it in two real browsers
and try a call before relying on it.

## Limits to know about

- The built-in interpreter is rule-based and English-only. It handles common phrasing well and will miss unusual
  phrasing, sarcasm, and long rambling statements; it attaches an item to the most plausible earlier one, which is
  sometimes wrong. Use "echo, move ..." or drag to fix, or configure a remote interpreter.
- The prototype is built for small rooms (up to 8). Video quality depends on each person's connection.
- Every change sends the whole canvas to everyone in the room, which is fine for hundreds of items but not
  thousands.
- Dragging is last-writer-wins; two people dragging the same item will fight.
