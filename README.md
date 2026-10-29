# Spar

Mock debugging interviews: open a broken codebase in the browser, talk through the bug with a live voice interviewer, fix it, and get a scored report.

Spar started as a hackathon project. The original build plan is in [IMPLEMENTATION.md](IMPLEMENTATION.md). It is a historical spec, so parts of it (for example, calling Gemini directly) no longer match the code.

## How it works

1. **Pick a scenario.** The client loads the catalog from `GET /api/scenarios`, and the user chooses a difficulty and then a scenario.
2. **Interview.** The Interview screen opens the scenario's visible files in Monaco and connects to `WS /api/interview/live`. The server builds a system prompt from the scenario metadata and visible files, then opens a Deepgram Voice Agent session (listen: `nova-3`, think: `gpt-5.4-mini`, speak: `aura-2-jupiter-en` by default). Mic audio (16 kHz linear16 PCM from an AudioWorklet) goes upstream, and the agent's 24 kHz PCM audio and transcript events come back to the browser.
3. **Tracking.** `useSessionTracker` records code edits, file switches, and transcript turns while the user works.
4. **Report.** When the user finishes, the client posts the tracker to `POST /api/reports`. The server builds a dossier (session metrics, final code, answer key) and runs four Backboard analyzer assistants in parallel: code behaviour, communication quality, interviewer dynamics, and bug-fix quality, each scored 0-10. A synthesizer assistant combines them into four dimensions, a verdict (`Strong Hire`, `Hire`, `Leaning Hire`, `Leaning No Hire`, or `No Hire`), and next steps. On first use the server creates the Backboard assistants, and after that it reuses them by name.

Answer keys never reach the browser. Files whose names contain `.fixed.` are treated as answer-key files, and the server strips them along with `bugLocation`, `bugLine`, `bugExplanation`, and `fixSummary` before sending scenarios to the client. The bug explanation and fix summary only come back inside the final report.

## Features

- **Scenario catalog**: 15 broken backend codebases (5 easy, 5 medium, 5 hard) defined in `spar-client/src/data/scenarios.js`. Examples include a null-user crash, duplicate orders, race conditions, N+1 queries, cache stampedes, and deadlocks. The server loads this file and serves it from `GET /api/scenarios`, with the hidden answer key stripped out.
- **In-browser editor**: Monaco editor with file tabs for the scenario's files.
- **Live voice interviewer**: the browser streams microphone PCM audio over a WebSocket (`/api/interview/live`). The server proxies it to the Deepgram Voice Agent API, which listens, thinks, and speaks back. You can also type answers instead of speaking.
- **Session tracking**: edits, file switches, and conversation turns are recorded while you work.
- **Report card**: `POST /api/reports` runs the session through Backboard assistants that score code behaviour, communication quality, interviewer dynamics, and bug-fix quality, then a synthesizer combines the scores. If synthesis fails, a local fallback report is built from the analyses.

## Tech stack

| Part | Tools |
|---|---|
| Client (`spar-client/`) | React 19, Vite, `@monaco-editor/react`, ESLint, plain inline styles and CSS |
| Server (`spar-server/`) | Node.js, Express 5, `ws`, `dotenv`, `cors`, `node:test` |
| External services | [Deepgram](https://deepgram.com) (Voice Agent and streaming STT), [Backboard](https://app.backboard.io) (LLM assistants for reports) |

## Project structure

```
spar-client/            React + Vite front end
  src/App.jsx           Screen router: landing → difficulty → scenario → interview → report
  src/screens/          Landing, DifficultySelect, ScenarioSelect, Interview, ReportLoading, Report
  src/components/       Timer, VoiceBar
  src/hooks/            useLiveVoiceInterview (voice WebSocket), useSessionTracker
  src/workers/          AudioWorklet that captures 16 kHz PCM from the mic
  src/lib/              API/WebSocket URL helpers, Monaco setup, report theme, demo report
  src/data/scenarios.js Scenario catalog (also read by the server)
spar-server/            Express + WebSocket back end
  index.js              Routes, Deepgram proxy, report pipeline
  lib/                  Backboard client, assistant specs, scenario loader, report helpers
  public/test-voice.html Manual voice-agent test page (served at /test-voice.html)
  test/                 node:test unit tests
IMPLEMENTATION.md       Original hackathon build plan (historical)
```

### Server endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Configuration status (which API keys are set, which models are in use) |
| GET | `/api/scenarios` | Client-safe scenario list |
| WS | `/api/interview/live` | Live voice interview (Deepgram Voice Agent proxy) |
| POST / DELETE | `/api/interview/sessions[/:id[/messages]]` | Text-based interview session API |
| WS | `/deepgram` | Raw Deepgram streaming transcription proxy |
| POST | `/api/reports` | Generate the report card |

### Live interview WebSocket protocol

Binary frames carry audio in both directions. JSON control messages:

| Direction | `type` | Notes |
|---|---|---|
| Client → server | `StartSession` | `{ scenarioId }`. Opens the Deepgram Voice Agent session. |
| Client → server | `InjectUserMessage` | `{ content }`. Typed answer, used instead of speech. |
| Client → server | `FinishSession` | Closes the agent session. |
| Server → client | `SessionReady` | `{ sessionId }`. The agent is ready to talk. |
| Server → client | `ConversationText`, `UserStartedSpeaking`, `AgentThinking`, `AgentStartedSpeaking`, `AgentAudioDone`, `InjectionRefused`, `Warning`, `Error` | Forwarded from Deepgram, or generated by the server for errors. |

## Prerequisites

- Node.js 20.19+ or 22.12+ (required by Vite 8) and npm
- A Deepgram API key (needed for interviews)
- A Backboard API key (needed for reports)
- A Chromium-based browser with microphone access, for voice

## Setup

```bash
git clone https://github.com/zhao0524/Spar.git
cd Spar

# Server
cd spar-server
npm install
cp .env.example .env   # then fill in DEEPGRAM_API_KEY and BACKBOARD_API_KEY
npm start              # http://localhost:3001

# Client (in a second terminal)
cd spar-client
npm install
npm run dev            # http://localhost:5173
```

In development, the Vite dev server proxies `/api` and `/deepgram` to `http://localhost:3001`. When the page is served on port 5173, the client connects WebSockets straight to port 3001.

## Environment variables

### `spar-server/.env`

| Variable | Purpose |
|---|---|
| `DEEPGRAM_API_KEY` | Deepgram key for the voice agent and transcription. Interviews return an error without it. |
| `BACKBOARD_API_KEY` | Backboard key for the report assistants. Reports return an error without it. |
| `BACKBOARD_BASE_URL` | Backboard API base URL (default `https://app.backboard.io/api`) |
| `BACKBOARD_LIVE_PROVIDER` / `BACKBOARD_LIVE_MODEL` | Reported by `/health` only; the code does not otherwise use these (default `google` / `gemini-2.5-flash`) |
| `BACKBOARD_REPORT_PROVIDER` / `BACKBOARD_REPORT_MODEL` | Provider and model for the report assistants (default `google` / `gemini-2.5-flash`) |
| `PORT` | Server port (default `3001`) |
| `CLIENT_ORIGIN` | Allowed CORS origin (default `http://localhost:5173`) |
| `DEEPGRAM_AGENT_THINK_MODEL`, `DEEPGRAM_AGENT_THINK_REASONING_MODE`, `DEEPGRAM_AGENT_THINK_TEMPERATURE` | Optional overrides for the voice agent's reasoning model |
| `DEEPGRAM_AGENT_LISTEN_MODEL`, `DEEPGRAM_AGENT_LISTEN_VERSION`, `DEEPGRAM_AGENT_LISTEN_LANGUAGE` | Optional overrides for the voice agent's speech-to-text |
| `DEEPGRAM_AGENT_SPEAK_MODEL` | Optional override for the voice agent's text-to-speech voice |
| `NODE_ENV` | Set to `production` to cache the scenario catalog instead of reloading it when the file changes |

### `spar-client/.env`

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | Optional. Set this only when the API is on a different origin from the client. |
| `VITE_CHECKOUT_BENCHMARK_REPORT` | Optional. Set to `true` to show a canned demo report for the Checkout Bug scenario instead of calling `POST /api/reports`. |

## Scripts

| Location | Command | Description |
|---|---|---|
| `spar-client` | `npm run dev` | Start the Vite dev server |
| `spar-client` | `npm run build` | Production build into `dist/` |
| `spar-client` | `npm run preview` | Serve the production build |
| `spar-client` | `npm run lint` | Run ESLint |
| `spar-server` | `npm start` / `npm run dev` | Start the server (`node index.js`) |
| `spar-server` | `npm test` | Run unit tests (`node --test`) |

## Testing

```bash
cd spar-server && npm test      # unit tests for the Backboard client, report pipeline, and scenario catalog
cd spar-client && npm run lint && npm run build
```

The server tests check that every scenario in `spar-client/src/data/scenarios.js` is valid: it has the required fields, a unique id, at least one visible file, and at least one `.fixed.` answer-key file. They also check that the client-facing catalog contains no answer-key data. Run them after you add or edit a scenario. The tests do not call Deepgram or Backboard. To check voice by hand, start the server and open `http://localhost:3001/test-voice.html`.

## Adding a scenario

Add an object to the `scenarios` array in `spar-client/src/data/scenarios.js` with `id`, `title`, `company`, `description`, `difficulty` (`easy` / `medium` / `hard`), `timeLimit` (seconds), `bugLocation`, `bugLine`, `bugExplanation`, `fixSummary`, and a `files` map. Include at least one buggy visible file and its corrected counterpart named with `.fixed.` (for example, `orders.js` and `orders.fixed.js`). In development the server reloads the file when it changes.

## Deployment

The root `vercel.json` builds the client on Vercel (`npm ci` and `npm run build` in `spar-client`, output `spar-client/dist`). Vercel serves only the static client. The Express/WebSocket server has to run on a host that supports long-lived WebSockets, and the Vercel project needs `VITE_API_BASE_URL` pointing at it.
