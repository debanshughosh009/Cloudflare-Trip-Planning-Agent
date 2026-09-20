# Technical Build Plan: Trip-Planning Agent on Cloudflare

**Audience:** a coding agent (Codex) that will implement this end-to-end.
**Base template:** `cloudflare/agents-starter`
**Chosen differentiator:** a **Trip-Planning Agent** — user describes a trip, the agent geocodes the destination, pulls a weather forecast, runs a multi-step Cloudflare Workflow to assemble a day-by-day itinerary, and persists the itinerary as agent state visible in a sidebar panel.

This theme was chosen deliberately because every external API it needs is **free and keyless** (Open-Meteo + OSM Nominatim), so the whole project can be built and deployed with zero third-party account setup beyond Cloudflare itself — maximizing automation and minimizing anything a human has to do manually.

If you (the human reviewer) want a different theme, only Section 5 (tool logic) and Section 7 (system prompt) need to change — the scaffolding, workflow wiring, state pattern, testing, and CI in every other section are theme-agnostic and should be reused as-is.

---

## 0. Assumptions and decisions made for you

- Package manager: `npm` (matches the starter's default; do not switch to pnpm/yarn unless the starter itself uses one — check `create-cloudflare` output before assuming).
- Language: TypeScript throughout (starter is already TS).
- LLM: Workers AI, default model already configured in the starter (`@cf/...` — confirm exact model string in the generated `server.ts`; do not hardcode a guessed model name, read it from the scaffolded file).
- No paid/keyed third-party services. If a tool needs external data, prefer a free/keyless API (Open-Meteo, Nominatim). Do not introduce a service that requires a signup unless explicitly told to.
- State storage: use the Agent's built-in SQLite-backed state (`this.state` / `this.setState`) — do not add D1/KV unless the built-in state proves insufficient (it won't for this scope).
- Multi-step coordination: implement the itinerary-building step as a **Cloudflare Workflow**, invoked from the Agent, to explicitly demonstrate the "workflow/coordination" requirement beyond just "it's a Durable Object."
- Voice input: **out of scope** for v1. Chat only. (Flag this as a stretch goal in the README, don't build it unless time remains after everything below is done and tested.)

---

## 1. Architecture overview

```
┌─────────────┐      WebSocket       ┌──────────────────────────┐
│  Browser UI │ ◄──────────────────► │  TripAgent (Durable Obj) │
│  (React,    │   useAgentChat()     │  extends AIChatAgent      │
│  from       │                      │                            │
│  starter)   │                      │  - onChatMessage()         │
└─────────────┘                      │  - tools: geocode,         │
                                      │    getForecast,            │
                                      │    planTrip (async tool)   │
                                      │  - state: TripState        │
                                      └───────────┬───────────────┘
                                                  │ triggers
                                                  ▼
                                      ┌──────────────────────────┐
                                      │  TripPlanningWorkflow     │
                                      │  (Cloudflare Workflows)   │
                                      │  step 1: geocode           │
                                      │  step 2: fetch forecast    │
                                      │  step 3: call LLM to draft │
                                      │          itinerary          │
                                      │  step 4: write result back │
                                      │          into Agent state  │
                                      └──────────────────────────┘
                                                  │
                                                  ▼
                                      ┌──────────────────────────┐
                                      │  Workers AI (LLM)          │
                                      │  Open-Meteo (weather)       │
                                      │  Nominatim (geocoding)      │
                                      └──────────────────────────┘
```

**Component mapping (for the README / reviewer table):**

| Required component | Implementation |
|---|---|
| LLM | Workers AI binding, called both directly in chat (`onChatMessage`) and from inside the Workflow step that drafts the itinerary |
| Workflow / coordination | `TripPlanningWorkflow` (Cloudflare Workflows) — multi-step, durable, retryable |
| User input | Chat UI from the starter (WebSocket, streaming) |
| Memory / state | `TripAgent.state.trip` — persisted itinerary, destination, dates, forecast; survives reconnect/refresh |

---

## 2. Repository structure (target state)

Start from what `create-cloudflare@latest --template cloudflare/agents-starter` generates, then add/modify these paths. Do not restructure the starter's existing layout beyond what's listed below — the goal is a minimal, reviewable diff.

```
agents-starter/
├── PROMPT_LOG.md                     # NEW — see Section 8
├── scripts/
│   └── log-prompt.mjs                # NEW — see Section 8
├── src/
│   ├── server.ts                     # MODIFIED — TripAgent class, tools, state
│   ├── tools.ts                      # MODIFIED — add geocode + forecast tools
│   ├── workflows/
│   │   └── trip-planning-workflow.ts # NEW — Cloudflare Workflow definition
│   ├── shared/
│   │   └── trip-types.ts             # NEW — shared TripState / DTO types
│   ├── client/
│   │   ├── App.tsx                   # MODIFIED — mount ItineraryPanel
│   │   └── components/
│   │       └── ItineraryPanel.tsx    # NEW — sidebar showing state.trip
│   └── tests/
│       ├── tools.test.ts             # NEW
│       └── workflow.test.ts          # NEW
├── wrangler.jsonc                    # MODIFIED — add workflow binding
├── .github/
│   └── workflows/
│       └── deploy.yml                # NEW — CI: test + deploy on push to main
├── README.md                         # MODIFIED — see Section 10
└── package.json                      # MODIFIED — add scripts (see Section 9)
```

> **Note to the coding agent:** the starter's actual file names for the chat server and tools file may differ slightly by version (e.g. `server.ts` vs `chat.ts`, `tools.ts` vs `tool-definitions.ts`). **Before writing any code, run the scaffold command and `view` the generated tree to confirm exact filenames**, then map this plan's steps onto the real files. Do not guess and create duplicate/conflicting files.

---

## 3. Environment & secrets

None required beyond Cloudflare authentication itself.

- `wrangler login` (interactive) or `CLOUDFLARE_API_TOKEN` env var for CI.
- No `.dev.vars` entries needed — Open-Meteo and Nominatim require no API key.
- Confirm `wrangler.jsonc` already has an `ai` binding (it will, from the starter). Do not add a second one.

Add to `wrangler.jsonc` a Workflow binding (adjust binding/class names to match what you actually implement):

```jsonc
{
  // ...existing config from starter...
  "workflows": [
    {
      "binding": "TRIP_PLANNING_WORKFLOW",
      "name": "trip-planning-workflow",
      "class_name": "TripPlanningWorkflow"
    }
  ]
}
```

Update the generated `Env` type (wherever the starter defines it, typically `worker-configuration.d.ts` or inline in `server.ts`) to include:

```ts
interface Env {
  AI: Ai; // already present from starter
  TripAgent: DurableObjectNamespace; // already present from starter, may be named differently
  TRIP_PLANNING_WORKFLOW: Workflow; // NEW
}
```

---

## 4. Shared types

Create `src/shared/trip-types.ts`:

```ts
export interface TripStop {
  day: number;              // 1-indexed day of the trip
  summary: string;          // e.g. "Explore Old Town, visit the cathedral"
  weatherNote?: string;     // e.g. "Sunny, high 22°C — good day for walking"
}

export interface TripState {
  destination: string | null;
  startDate: string | null;      // ISO date
  numDays: number | null;
  coordinates: { lat: number; lon: number } | null;
  forecast: DailyForecast[] | null;
  itinerary: TripStop[] | null;
  status: "idle" | "planning" | "ready" | "error";
  error?: string;
}

export interface DailyForecast {
  date: string;         // ISO date
  tempMaxC: number;
  tempMinC: number;
  precipitationProbability: number; // 0-100
  conditionCode: number; // WMO weather code from Open-Meteo
}

export const initialTripState: TripState = {
  destination: null,
  startDate: null,
  numDays: null,
  coordinates: null,
  forecast: null,
  itinerary: null,
  status: "idle",
};
```

This file is imported by both `server.ts` (Agent) and `workflows/trip-planning-workflow.ts`, and by the client `ItineraryPanel.tsx` — keep it dependency-free (no Cloudflare-specific imports) so it can be safely imported client-side.

---

## 5. Tools implementation

Add to (or create, if the starter's tool file doesn't already exist under this name) `src/tools.ts`. Follow whatever tool-definition pattern the starter already uses (likely `ai` SDK's `tool()` helper with a Zod schema) — **inspect an existing tool in the starter first and match its exact shape**, don't invent a different pattern.

### 5.1 `geocodeDestination` tool

```ts
import { z } from "zod";
import { tool } from "ai"; // confirm this import path matches the starter's existing tools

export const geocodeDestination = tool({
  description:
    "Look up latitude/longitude for a place name using OpenStreetMap Nominatim. Use this before fetching weather or planning an itinerary.",
  parameters: z.object({
    place: z.string().describe("City or place name, e.g. 'Lisbon, Portugal'"),
  }),
  execute: async ({ place }) => {
    const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(
      place
    )}&format=json&limit=1`;
    const res = await fetch(url, {
      headers: { "User-Agent": "cloudflare-trip-agent/1.0 (assignment project)" },
    });
    if (!res.ok) throw new Error(`Geocoding failed: ${res.status}`);
    const data = (await res.json()) as Array<{ lat: string; lon: string; display_name: string }>;
    if (!data.length) throw new Error(`No results for "${place}"`);
    return {
      lat: parseFloat(data[0].lat),
      lon: parseFloat(data[0].lon),
      resolvedName: data[0].display_name,
    };
  },
});
```

> Nominatim's usage policy requires a descriptive `User-Agent` and reasonable rate limits (max ~1 req/sec). This is a low-traffic demo app so default fetch behavior is fine; do not add retry-storm logic.

### 5.2 `getWeatherForecast` tool

```ts
export const getWeatherForecast = tool({
  description:
    "Get a multi-day daily weather forecast for given coordinates using Open-Meteo (no API key required).",
  parameters: z.object({
    lat: z.number(),
    lon: z.number(),
    days: z.number().min(1).max(16).default(5),
  }),
  execute: async ({ lat, lon, days }) => {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max,weathercode&forecast_days=${days}&timezone=auto`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`Forecast fetch failed: ${res.status}`);
    const data = (await res.json()) as {
      daily: {
        time: string[];
        temperature_2m_max: number[];
        temperature_2m_min: number[];
        precipitation_probability_max: number[];
        weathercode: number[];
      };
    };
    return data.daily.time.map((date, i) => ({
      date,
      tempMaxC: data.daily.temperature_2m_max[i],
      tempMinC: data.daily.temperature_2m_min[i],
      precipitationProbability: data.daily.precipitation_probability_max[i],
      conditionCode: data.daily.weathercode[i],
    }));
  },
});
```

### 5.3 `planTrip` tool (kicks off the Workflow)

This tool does **not** do the planning itself — it triggers the Workflow and returns immediately, demonstrating async/durable coordination rather than doing everything inline.

```ts
export function makePlanTripTool(env: Env, agentId: string) {
  return tool({
    description:
      "Kick off full multi-day itinerary planning for a destination. This runs as a background workflow — inform the user it may take a few seconds and that the itinerary will appear in the sidebar when ready.",
    parameters: z.object({
      destination: z.string(),
      startDate: z.string().describe("ISO date, e.g. 2026-10-01"),
      numDays: z.number().min(1).max(14),
    }),
    execute: async ({ destination, startDate, numDays }) => {
      await env.TRIP_PLANNING_WORKFLOW.create({
        params: { destination, startDate, numDays, agentId },
      });
      return {
        status: "started",
        message: `Started planning a ${numDays}-day trip to ${destination}. Check the itinerary panel shortly.`,
      };
    },
  });
}
```

> This tool needs `env` and the agent's own id at call time, so it must be constructed inside the Agent class (where both are available) rather than exported as a static tool like the other two. Check how the starter's existing tools that need `this.env` are structured (there should be at least one, e.g. the scheduling tool) and mirror that pattern exactly.

---

## 6. Cloudflare Workflow

Create `src/workflows/trip-planning-workflow.ts`:

```ts
import { WorkflowEntrypoint, WorkflowStep, WorkflowEvent } from "cloudflare:workers";
import type { TripState, DailyForecast, TripStop } from "../shared/trip-types";

type Params = {
  destination: string;
  startDate: string;
  numDays: number;
  agentId: string;
};

export class TripPlanningWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    const { destination, startDate, numDays, agentId } = event.payload;

    const geo = await step.do("geocode", async () => {
      const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(
        destination
      )}&format=json&limit=1`;
      const res = await fetch(url, {
        headers: { "User-Agent": "cloudflare-trip-agent/1.0 (assignment project)" },
      });
      const data = (await res.json()) as Array<{ lat: string; lon: string }>;
      if (!data.length) throw new Error(`No geocoding result for ${destination}`);
      return { lat: parseFloat(data[0].lat), lon: parseFloat(data[0].lon) };
    });

    const forecast: DailyForecast[] = await step.do("fetch-forecast", async () => {
      const url = `https://api.open-meteo.com/v1/forecast?latitude=${geo.lat}&longitude=${geo.lon}&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max,weathercode&forecast_days=${numDays}&timezone=auto`;
      const res = await fetch(url);
      const data = (await res.json()) as {
        daily: {
          time: string[];
          temperature_2m_max: number[];
          temperature_2m_min: number[];
          precipitation_probability_max: number[];
          weathercode: number[];
        };
      };
      return data.daily.time.map((date, i) => ({
        date,
        tempMaxC: data.daily.temperature_2m_max[i],
        tempMinC: data.daily.temperature_2m_min[i],
        precipitationProbability: data.daily.precipitation_probability_max[i],
        conditionCode: data.daily.weathercode[i],
      }));
    });

    const itinerary: TripStop[] = await step.do("draft-itinerary", async () => {
      const prompt = `You are planning a ${numDays}-day trip to ${destination} starting ${startDate}.
Here is the daily forecast: ${JSON.stringify(forecast)}.
For each day, suggest one concise activity plan (1-2 sentences) that makes sense given the weather
(e.g. indoor activities on rainy days). Respond ONLY as a JSON array of objects:
[{ "day": 1, "summary": "...", "weatherNote": "..." }, ...] with no extra text.`;

      const result = await this.env.AI.run("@cf/meta/llama-3.3-70b-instruct-fp8-fast", {
        messages: [{ role: "user", content: prompt }],
      });

      const text = (result as { response?: string }).response ?? "[]";
      const cleaned = text.replace(/```json|```/g, "").trim();
      try {
        return JSON.parse(cleaned) as TripStop[];
      } catch {
        // Fallback: at least return the forecast-based skeleton if the model
        // didn't return valid JSON, so the workflow doesn't hard-fail.
        return forecast.map((f, i) => ({
          day: i + 1,
          summary: "Explore the area (auto-generated fallback — model output was not valid JSON).",
          weatherNote: `${f.tempMinC}–${f.tempMaxC}°C, ${f.precipitationProbability}% chance of rain`,
        }));
      }
    });

    await step.do("write-back-to-agent", async () => {
      const id = this.env.TripAgent.idFromString(agentId); // confirm exact binding name matches server.ts
      const stub = this.env.TripAgent.get(id);
      // Use whatever RPC/callable method pattern the Agent exposes for external writes —
      // e.g. a @callable() method like `applyPlannedTrip(state: Partial<TripState>)`.
      await stub.applyPlannedTrip({
        coordinates: geo,
        forecast,
        itinerary,
        status: "ready",
      });
    });
  }
}
```

> **Confirm exact Workflow API surface** (`WorkflowEntrypoint`, `step.do`, `WorkflowEvent`) against the current Cloudflare Workflows docs (`https://developers.cloudflare.com/workflows/`) before writing this file for real — API shape has been stable but always verify against the live docs rather than trusting this snippet verbatim, since SDK signatures can shift between versions.
> **Confirm the LLM model string** by checking what the scaffolded `server.ts` already uses in `onChatMessage`, and reuse that exact string here rather than hardcoding a possibly-stale model name.

---

## 7. Agent class wiring

Modify the generated Agent class (likely in `server.ts`) along these lines — again, match the starter's actual class/method names, this is illustrative:

```ts
import { AIChatAgent } from "some-starter-import"; // match existing import
import { initialTripState, type TripState } from "./shared/trip-types";
import { geocodeDestination, getWeatherForecast, makePlanTripTool } from "./tools";

export class TripAgent extends AIChatAgent<Env, TripState> {
  initialState = initialTripState;

  async onChatMessage() {
    const tools = {
      geocodeDestination,
      getWeatherForecast,
      planTrip: makePlanTripTool(this.env, this.name), // this.name / this.id depending on starter's API
    };
    // ... call the LLM with these tools merged into whatever the starter already passes,
    // following the exact streaming pattern already present in onChatMessage.
  }

  // Called by the Workflow's write-back step.
  async applyPlannedTrip(update: Partial<TripState>) {
    this.setState({ ...this.state, ...update });
  }
}
```

**System prompt** — update the existing system prompt string to something like:

```
You are a trip-planning assistant. When the user describes a destination, dates, and
trip length, first geocode the destination, then decide whether to fetch a quick
forecast yourself or call planTrip to generate a full day-by-day itinerary in the
background. Always tell the user when planTrip has been started and that results
will appear in the itinerary panel. Keep responses concise and friendly.
```

---

## 8. Client UI

Add `src/client/components/ItineraryPanel.tsx`:

```tsx
import type { TripState } from "../../shared/trip-types";

export function ItineraryPanel({ trip }: { trip: TripState }) {
  if (trip.status === "idle") {
    return <div className="itinerary-panel">No trip planned yet — ask the agent to plan one!</div>;
  }
  if (trip.status === "planning") {
    return <div className="itinerary-panel">Planning your trip to {trip.destination}…</div>;
  }
  if (trip.status === "error") {
    return <div className="itinerary-panel itinerary-panel--error">Something went wrong: {trip.error}</div>;
  }
  return (
    <div className="itinerary-panel">
      <h2>{trip.destination}</h2>
      <ul>
        {trip.itinerary?.map((stop) => (
          <li key={stop.day}>
            <strong>Day {stop.day}:</strong> {stop.summary}
            {stop.weatherNote && <div className="weather-note">{stop.weatherNote}</div>}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Wire it into the starter's main layout (`App.tsx` or equivalent) alongside the chat window, reading `agent.state` from whatever hook the starter already uses to subscribe to Agent state updates (there should be one already, since state sync is a built-in starter feature — do not build a new subscription mechanism).

---

## 9. Testing

### 9.1 Unit tests — `src/tests/tools.test.ts`

- Mock `fetch` (use `vi.stubGlobal("fetch", ...)` with Vitest) and assert:
  - `geocodeDestination` parses a Nominatim-shaped response correctly and throws on empty results
  - `getWeatherForecast` maps Open-Meteo's daily arrays into the `DailyForecast[]` shape correctly for a 3-day mock response

### 9.2 Workflow test — `src/tests/workflow.test.ts`

- If the starter's test setup supports invoking Workflow steps directly (check `@cloudflare/vitest-pool-workers` docs for Workflow testing support), write a test that runs `TripPlanningWorkflow.run()` with mocked `fetch` and mocked `this.env.AI.run`, and asserts `applyPlannedTrip` is called with a well-formed `TripStop[]`.
- If direct Workflow unit testing isn't well-supported yet, fall back to testing the three step functions as extracted, exported plain functions instead of testing them through `step.do` — refactor the workflow file to export `geocodeStep`, `forecastStep`, `draftItineraryStep` as standalone functions the class calls, so they're independently testable regardless of Workflow test tooling maturity.

### 9.3 Manual test checklist (run after `npm run dev`)

- [ ] Ask the agent to plan a 3-day trip to a real city → confirm chat responds it's started planning
- [ ] Confirm the itinerary panel updates within ~10-20 seconds without a page refresh
- [ ] Refresh the page mid-conversation → confirm state (itinerary + chat history) persists
- [ ] Ask about a nonsense destination (e.g. "Xyzzyplex") → confirm graceful error handling (chat says it couldn't find that place, `status` becomes `"error"`, no unhandled exception in the console)
- [ ] Ask for weather only (no full plan) → confirm the agent uses `getWeatherForecast` directly without triggering the Workflow

Add to `package.json`:
```json
"scripts": {
  "test": "vitest run",
  "test:watch": "vitest"
}
```

---

## 10. Automation: prompt logging

Automate prompt-history capture so nothing has to be manually written up after the fact.

Create `scripts/log-prompt.mjs`:

```js
#!/usr/bin/env node
// Usage: npm run log-prompt -- "the prompt text I just used"
import { appendFileSync } from "node:fs";

const prompt = process.argv.slice(2).join(" ");
if (!prompt) {
  console.error("Usage: npm run log-prompt -- \"<prompt text>\"");
  process.exit(1);
}
const timestamp = new Date().toISOString();
appendFileSync(
  "PROMPT_LOG.md",
  `\n## ${timestamp}\n\n${prompt}\n`
);
console.log("Logged prompt to PROMPT_LOG.md");
```

Add to `package.json`:
```json
"scripts": {
  "log-prompt": "node scripts/log-prompt.mjs"
}
```

If using an agent tool that keeps its own transcript (e.g. Claude Code, Codex CLI session logs), the simplest automation is: **at the end of the session, export or copy the full transcript into `PROMPT_LOG.md` in one pass**, rather than logging turn-by-turn. Prefer that over the manual script above if the tool supports a transcript export — it's more complete and less error-prone. Note in the README which method was used.

Initialize `PROMPT_LOG.md` with a one-line header:
```markdown
# Prompt History

This file records the AI-assisted prompts used to build this project.
```

---

## 11. CI/CD automation

Create `.github/workflows/deploy.yml`:

```yaml
name: Test and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - name: Deploy to Cloudflare
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

This means: after the coding agent finishes, the human only needs to (a) push to `main` and (b) set the `CLOUDFLARE_API_TOKEN` repo secret once — everything else (test, deploy) is automatic from then on.

---

## 12. package.json scripts (consolidated)

Ensure the final `package.json` has at least:

```json
{
  "scripts": {
    "dev": "... (unchanged from starter)",
    "deploy": "... (unchanged from starter)",
    "test": "vitest run",
    "test:watch": "vitest",
    "log-prompt": "node scripts/log-prompt.mjs",
    "check": "tsc --noEmit && npm run test"
  }
}
```

Run `npm run check` before every commit as a cheap local CI gate.

---

## 13. README.md content requirements

The final README must include, in this order:

1. One-paragraph description of the Trip-Planning Agent
2. The component-mapping table from Section 1 of this plan
3. Setup/run instructions (`npx create-cloudflare...` through `npm run dev`)
4. Architecture diagram (reuse Section 1's diagram, reformatted as needed)
5. A "Known limitations / stretch goals" section — explicitly mention voice input was scoped out, and any Workflow-testing caveats from Section 9.2 if the fallback approach was used
6. A link/reference to `PROMPT_LOG.md`

---

## 14. Definition of done (acceptance checklist)

Do not consider this complete until every item below is true:

- [ ] `npm run dev` runs locally with no errors, chat works end-to-end
- [ ] Asking the agent to plan a trip triggers the Workflow (visible in `wrangler dev` logs or Cloudflare dashboard) and the itinerary panel updates without a manual refresh
- [ ] State survives a hard page refresh mid-session
- [ ] `npm run test` passes with the tool + workflow-step tests from Section 9
- [ ] `npm run check` (typecheck + tests) passes clean
- [ ] `npm run deploy` succeeds and the deployed URL reproduces the same behavior as local dev
- [ ] `PROMPT_LOG.md` exists and contains a real, non-fabricated record of the prompts used
- [ ] README fully covers Section 13's requirements
- [ ] CI (`.github/workflows/deploy.yml`) is green on the `main` branch
- [ ] No unused starter boilerplate left in (leftover sample tools like weather/timezone from the stock starter should be removed or clearly repurposed, not left dangling unused)

---

## 15. Explicit non-goals for this pass

To keep scope bounded, do **not** build these unless everything in Section 14 is done and there's clear remaining time:

- Voice input/output
- Multi-user auth or per-user accounts (single-session-per-agent is fine)
- A production-grade design system for the UI — functional and clean is sufficient, don't over-invest in styling
- Persisting trips across multiple separate agent instances / a "trip history list" feature — one active trip per agent session is sufficient for this assignment
