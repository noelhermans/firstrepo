# AgentOS ("Agentic OS — Mission Control") — Technical Teardown

**Target:** https://agentos.maddogmurdockai.com/
**Date of analysis:** 2026-11-05
**Method:** I read the server-rendered HTML of the home page and all 50+ routes, downloaded and statically analyzed every JavaScript and CSS chunk the pages reference (~3.6 MB of JS, ~410 KB of CSS), checked the HTTP response headers, and made a handful of read-only `GET` requests to public API endpoints. **No `POST`, `PUT` or `DELETE` requests were sent**, and nothing was authenticated, executed or modified on the target.

> **Confidence legend used below**
> - **Observed:** seen directly in headers, HTML, bundle code or an API response.
> - **Inferred:** a strong conclusion from observed evidence, not directly visible.
> - **Claimed:** text the UI itself shows (marketing copy, model names). Reported as written, not verified.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Provenance: where this comes from](#2-provenance-where-this-comes-from)
3. [Hosting and deployment](#3-hosting-and-deployment)
4. [Frontend stack](#4-frontend-stack)
5. [Information architecture: every page](#5-information-architecture-every-page)
6. [Backend: the Next.js API layer](#6-backend-the-nextjs-api-layer)
7. [Core integration patterns (the "how")](#7-core-integration-patterns-the-how)
   - 7.1 Agent adapter / "agent room" pattern
   - 7.2 Chat streaming protocol (NDJSON over fetch)
   - 7.3 Claude Code integration (stream-json)
   - 7.4 Workspace + live preview pattern
   - 7.5 Web terminal (Herdr) over SSE + POST
   - 7.6 Voice agent "Apollo" (OpenAI Realtime over WebRTC)
   - 7.7 Multi-agent room (Mastermind)
   - 7.8 Councils / boardrooms (Fusion, Sakana, MoA)
   - 7.9 Loop engineering (builder + adversarial judge)
   - 7.10 Pipeline (inbox → shipped)
   - 7.11 Memory layer (Obsidian vault + Omi)
   - 7.12 Health / vitals
   - 7.13 Scheduled / background jobs
8. [Agent and model inventory](#8-agent-and-model-inventory)
9. [Visual design system](#9-visual-design-system)
10. [Security assessment](#10-security-assessment)
11. [How to build your own](#11-how-to-build-your-own)
12. [Platform-grade variant (k8s / AKS / OpenShift / k3s)](#12-platform-grade-variant-k8s--aks--openshift--k3s)
13. [Open questions before a detailed design](#13-open-questions-before-a-detailed-design)
14. [Appendix A — full API endpoint inventory](#appendix-a--full-api-endpoint-inventory)
15. [Appendix B — raw evidence snippets](#appendix-b--raw-evidence-snippets)
16. [Sources](#sources)

---

## 1. Executive summary

- **What it is:** a **single-user, self-hosted "mission control" web app** that puts one UI over a large collection of AI agent CLIs, model gateways and content tools installed on the owner's own computer. It is *not* a multi-tenant SaaS product.
- **The core trick:** it implements **almost no agent logic itself.** Each "agent" is an existing CLI or service: Claude Code, OpenAI Codex, OpenClaw, Hermes (Nous Research), OpenCode, Kimi CLI, Grok Build CLI, Ollama, OpenRouter, and more. Next.js route handlers on the same machine spawn or call these and **stream their output back to the browser**.
- **Shared memory** is an **Obsidian vault on disk**: plain markdown files. Every conversation is "auto-saved to Obsidian", and the vault is read back in as context for agents. Omi (a wearable) memories are pulled in too.
- **Recurring UX pattern:** *prompt → agent writes files into a workspace folder → the UI serves those files back and renders them live in a sandboxed iframe.* Almost every "coder" page is a variation of this.
- **The most sophisticated pieces:**
  1. **"Apollo" voice agent:** OpenAI Realtime over **WebRTC**, with server-minted ephemeral keys and tool calls routed back into the local API.
  2. **Browser terminal** (Herdr) carried over **SSE down / POST up**.
  3. **Multi-agent group chat** and "boardroom" councils.
- **Stack:** Next.js App Router (Turbopack build, prerendered), React, three.js with custom GLSL shaders, cmdk, vaul, motion, lucide, Radix, an xterm.js-style terminal, Monaco, and likely Tailwind v4 plus a lot of hand-written CSS. It runs as a **self-hosted `next start`** server behind **Cloudflare**.
- **Biggest lesson:** the API appears to be **exposed to the internet without authentication** (§10). For a web UI that can spawn shells and agents on the host, that is the main thing *not* to copy.

---

## 2. Provenance: where this comes from

- **Inferred:** the app comes out of **Julian Goldie's "Agent OS" content series** (May–July 2026), shared in his "AI Profit Boardroom" community.
  - Secondary summaries of those videos describe a locally hosted Next.js mission-control dashboard built in a single Claude Desktop session "without writing any code".
  - They describe agent conversations logged to an Obsidian vault and fed back as context.
  - They describe a three-layer model: **Claude = brain, Hermes = execution, OpenClaw = connection.**
  - Features they mention (session analytics, skills inventory, kanban, goal tracker, content studio) all appear in this app.
- **Observed** evidence linking this instance to that lineage:
  - `/api/seo/published` lists SEO "network" sites under Goldie's brands, e.g. `bestaiagentcommunity.com`, `aiprofitboardroom.com` and `juliangoldieaiautomation.com`.
  - The Sakana page says "#1 on goldiebench".
  - The meta description is "Your command center for Claude, OpenClaw, Hermes", which matches the video framing.
- **Observed:** `/api/version` returns `{"version":"2026-10-18","userName":"<owner first name>"}`. The instance is **one person's customized copy** of the template, version-dated 2026-10-18.
- **Inferred:** the UI copy says "on your Mac" everywhere (e.g. "100% on your Mac", `~/.local/bin/herdr`, `~/Developer/awesome-llm-apps`), but `/api/vitals` reports **Windows paths** (`C:\Users\...`) for the OpenClaw session store. Most likely the template was written on macOS and this copy runs on Windows (or the OpenClaw gateway runs on a separate Windows host). The copy text was never adapted.
- The many hyper-specific pages (Ad Studio, Rank-and-Rent website generator, SEO "parasite" tooling, Sales Agent OS, Shorts, Thumbnails) show this is a **content-creator / marketer's workbench**, extended page by page by asking an agent to "add X to my Agent OS".

---

## 3. Hosting and deployment

### 3.1 Response headers (observed, from `GET /`)

```
HTTP/2 200
content-type: text/html; charset=utf-8
cache-control: s-maxage=31536000
vary: rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch, Accept-Encoding
x-nextjs-cache: HIT
x-nextjs-prerender: 1
x-nextjs-stale-time: 300
x-powered-by: Next.js
server: cloudflare
cf-cache-status: DYNAMIC
nel / report-to: Cloudflare NEL endpoints
alt-svc: h3=":443"
```

### 3.2 Interpretation

| Signal | Meaning |
|---|---|
| `x-powered-by: Next.js`, `x-nextjs-cache: HIT`, `x-nextjs-prerender: 1` | A **self-hosted Next.js server** (`next start` or standalone output) serving statically prerendered App Router pages from its own ISR cache. |
| No `x-vercel-*` headers, no `x-opennext`, no Workers signatures | **Not Vercel, and not Cloudflare Workers/OpenNext.** The Next server is the origin. |
| `server: cloudflare`, `cf-cache-status: DYNAMIC` | Cloudflare proxies the hostname but **does not cache the HTML** (DYNAMIC = not eligible for caching under current rules). |
| Windows paths in `/api/vitals`, CLIs logged in with consumer subscriptions | The origin is the **owner's personal machine**. |
| **Inferred:** origin reachability | A home PC behind NAT plus a Cloudflare-proxied subdomain almost always means a **Cloudflare Tunnel (`cloudflared`)** pointing at `localhost:3000`. |
| `vary: rsc, next-router-*` | Standard App Router RSC/prefetch negotiation, Next 15+ era. |
| `turbopack-*.js` runtime chunk, `~`-style hashed chunk names | Built with **Turbopack** (`next build --turbopack`), i.e. a recent Next.js (15.x late / 16.x). |

### 3.3 Other observations

- No `manifest.webmanifest` or `manifest.json` (both 404), yet the app declares `mobile-web-app-capable`, `apple-mobile-web-app-title="Agentic OS"`, `apple-mobile-web-app-status-bar-style="black-translucent"` and `viewport-fit=cover`. It's built to be **"Add to Home Screen"-able on iOS**: a phone remote for the home PC. The mobile bottom nav (Mission / Claude / Memory / Agents / Menu, plus Hermes / Apollo buttons) confirms this.
- `robots.txt` exists (200).
- `theme-color: #15101a`, a dark aubergine, the default theme background.
- Viewport sets `maximum-scale=1`, which disables pinch-zoom (an app-like feel, at an accessibility cost).

---

## 4. Frontend stack

### 4.1 Framework and build

- **Next.js App Router** with **React** (the React DOM internals are in the bundle). Pages are **prerendered** and hydrate as client components. Almost all interactivity is client-side `fetch` to `/api/*`.
- **Turbopack** build output (`/_next/static/chunks/turbopack-*.js`, roughly 75 hashed chunks across all routes).
- **Fonts:**
  - Two self-hosted `next/font` woff2 files (preloaded).
  - Google Fonts: **Bricolage Grotesque** (display), **Manrope** (UI text), **JetBrains Mono** (code/terminal), **Caveat** (handwritten accents, e.g. "click the core ✦").

### 4.2 Libraries identified in the bundle

| Library | Evidence | Used for |
|---|---|---|
| **three.js** | `WebGLRenderer({alpha:!0,antialias:!0,powerPreference:"low-power"})`, `ShaderMaterial`, `IcosahedronGeometry(1,64)`, `Points`, `AdditiveBlending`, inline GLSL (`gl_FragColor = vec4(vC, smoothstep(.5,0.,d)*vA)`) | The animated "core" orb on Mission Control (a displaced icosphere shader), particle fields and starfields; "Galaxy ✦" memory view |
| **cmdk** | module present | ⌘K command palette |
| **vaul** | module present | Drawers / bottom sheets (mobile) |
| **motion** (Framer Motion successor) | module present | UI animation |
| **lucide-react** | icon path arrays `["path",{d:"…",key:"…"}]` | Icons |
| **Radix UI** | `radix-ui.com` reference | Primitives (dialogs, popovers) |
| **xterm.js** (or a compatible fork) | `t.loadAddon(x)`, `t.open(el)`, `x.fit()`, `t.onData`, `t.onResize`, `t.write(...)`, terminal theme object | Herdr / jcode / opencode / Prime terminals |
| **Monaco editor** | module present | Code view in workspaces |
| **Tailwind CSS v4** (likely) | `--color-amber-200:#fee685`-style theme tokens; very few `--tw-` variables | Utility CSS, mixed with lots of hand-written CSS and custom properties |

**Not found:** no global state library (zustand, jotai, redux), no SWR or TanStack Query, no Socket.IO, no tRPC, no auth library (NextAuth/Auth.js, Clerk) and no analytics SDK. State is local `useState`/`useRef`/`useCallback`, and data fetching is plain `fetch(..., {cache:"no-store"})`, often polled.

### 4.3 App shell

- **Left sidebar** with sections:
  - **Workspace:** Mission Control
  - **Agent Orchestration:** Herdr, Paperclip, AI Agent Mastermind
  - **Pipeline:** Agent Kanban
  - **Agents:** the long list in §8
  - **Self / Create / Studio / Growth:** Sales, Ad Studio, Ads, Loop, SEO, 3D Worlds, Website Generator, Design OS, Open Design, Video, OpenMontage, Video Editor, Music, Game Studio, App Lab, Thumbnails, Notebook, Kanban, Memory
  - **Footer:** "Your workspace: claude · openclaw · hermes" with an "Obsidian vault" link.
- **Top bar:** breadcrumb ("Workspace / Mission Control"), page title and tagline, a "Local · Studio" mode indicator, a **⌘K** palette trigger and an "All systems" status pill (fed by `/api/vitals`).
- **Mobile bottom nav:** Mission · Claude · Memory · Agents · Menu, plus floating **Hermes** / **Apollo** (voice) buttons and a "Full view" toggle.
- **Per-agent pages** share a template: section label ("Agent · Claude"), title, one-line description, then **tabs** (Chat / Workspace / Studio / Control Room / Sessions …), a "New conversation" button, the composer ("to send · Send") and the footer note "**Auto-saved to Obsidian**". Slow agents show expectation hints such as "openclaw: 20–40s (ollama/deepseek-v4-flash) — keep waiting" or "antigravity: 10–90s".

### 4.4 Mission Control (home page) widgets

- **Greeting + clock:** "Good morning." / `--:--:--`, then "Waking up your agents…" while health checks run.
- **Agent constellation** around the 3D core: Claude, Hermes, Codex, OpenClaw, Antigravity ("click the core ✦"). The "— /5 agents online" counter comes from `/api/vitals`.
- **KPI tiles:** SEO posts this week · radar signals · things built.
- **Quick-launch cards:**
  - Hermes Chat ("Your main agent")
  - Apollo Voice ("Talk, don't type")
  - Oracle Radar ("Today's AI news")
  - SEO Machine ("Rank the network")
  - Video Studio ("Cut, caption, render")
  - Thumbnails ("gpt-image-2 concepts")
  - Builds ("Everything you've built")
  - NotebookLM ("Shorts, podcasts, decks")
- **Oracle Radar panel:** "WHAT TO MAKE TODAY", fed by `/api/radar/latest`.
- **SEO network panel:** live posts / this week / today, fed by `/api/seo/published`.
- **Today's Focus:** a to-do list with history (`/api/todos`, `/api/todos?date=`).
- **Idea Pipeline counters:** inbox / shaping / building / review / shipped, with a "Capture" box (`/api/pipeline`, `/api/pipeline/capture`).
- **Morning briefing** (`/api/hermes/briefing`).

---

## 5. Information architecture: every page

Observed from each route's server-rendered HTML. Model and product names are **as claimed by the UI**.

| Route | Section | Title | What the UI says it does | Key tabs / controls |
|---|---|---|---|---|
| `/` | Workspace | Mission Control | "See what needs you. Direct what happens next." | See §4.4 |
| `/claude` | Agent | Claude | "Direct streaming channel to your Claude Code CLI. Voice in, auto-logged to Obsidian." | Chat · Workspace · Artifacts · **Ultracode** · **Ant CLI** · Agents · Shorts |
| `/hermes` | Agent | Hermes | "Nous Research agent. Sessions, skills, kanban — and a chat line." | Chat · Bots · **Apollo** · Hermes Oracle · Hermes Muse · Hermes Astros · Studio · Sessions · Lead generation · **Mixture** · Workspace · **MCPs** · Manage · Control Room · **Goal Mode**; model picker ("Step 5 · Free") |
| `/openclaw` | Agent | OpenClaw | "Chat one-shot or open the control room. Logged to your vault." | Chat · Studio · Workspace · Control Room; runs on `ollama/deepseek-v4-flash` |
| `/codex` | Agent | Codex | "OpenAI's coding agent. Chat, set long-running goals, preview anything it builds." | Chat · Goal Mode · Sessions · Workspace; "Codex · Direct", model "GPT-6 Astra · ChatGPT" |
| `/opencode` | Agent | OpenCode | "The open-source terminal coding agent, wired to whichever provider you point it at." | Terminal · Workspace; cwd `~/.agentic-os/opencode`; "75+ providers · free tier built in" |
| `/antigravity` | Agent | Antigravity | "Gemini CLI's successor. Go-based, multi-agent harness, plugins, async workflows." | Chat · Builds · Agent Teams (`/teamwork-preview`) |
| `/kimi` | Agent | Kimi Code | "Moonshot's Kimi coding agent — now with K3 (2.5T · 1M context)." | Chat · Workspace; "Every send runs `kimi -p … --output-format stream-json`" |
| `/glm` | Agent | GLM 5.2 | "Zhipu's GLM-5.2 flagship coder on the z.ai Coding Plan. 1M context." | Chat · Workspace |
| `/glm-code` | Agent | GLM Code | "GLM's coding agent on your z.ai plan." | Engine switch: GPT 5.6 (Sol) via Codex on ChatGPT login · GLM 5.2 · Kimi K3; Terminal · Workspace |
| `/prime` | Agent | Prime Agent | "Prime Intellect's self-improving harness… can't call the work done until the checks pass." | Terminal · Workspace · Sessions; cwd `~/.agentic-os/prime-agent/builds` |
| `/grok` | Agent | Grok Build | "xAI's Grok 4.7 coding agent … through the Grok Build CLI, signed in on your X Premium+ plan." | Model select 4.7 / 4.7 Fast / 4.6 / 4.5; Chat · Workspace |
| `/starnet` | Agent | StarNet | "Manage your agent crew from a living pixel-art space station… every object you place is a real tool grant." | Embedded app from `~/starnet`; Reload · Open full |
| `/openmausbot` | Agent | OpenMausBot | "A team of agents living in a chat app. Every contact is a real agent with its own model, memory and computer." | Team · Workspace · New bot · Contacts |
| `/freeclaude` | Agent | Free Claude Code | "Open-source proxy. Same Claude CLI, routed to your local model — free." | Agent Factory · Chat & Workspace; voice-to-build |
| `/omniroute` | Agent | Free AI Coder | "Four free coding engines: OmniRoute (free pool, zero keys), 9Router (your own logins), Ox Alpha, Step 5 on Nous Portal." | Engine cards with live status; Build · Preview · Code; Saved builds; Conversation history |
| `/hy3-coder` | Agent | HY3 Coder | "Tencent's 295B agentic model, free on OpenRouter." | Needs OpenRouter key |
| `/deepseek-coder` | Agent | DeepSeek Coder | "DeepSeek V4 Flash straight off DeepSeek's own API." | V4 Pro ↔ Flash |
| `/muse-code` | Agent | Muse Code | "Meta's Muse Wonder … via OpenRouter." | Model select |
| `/mimo-code` | Agent | MiMo Code | "Xiaomi's MiMo-V2.6 Pro + Flash — open-weight." | Pro / Flash |
| `/higgsfield` | Studio | Higgsfield | "Image and video generation over MCP — every asset, session and prompt kept in one gallery." | Campaign studio |
| `/fusion` | Agent | Fusion Boardroom | "OpenRouter Fusion — a panel of models deliberates with web search, then a judge writes the verdict." | Boardroom · Workspace; presets: SEO content council, Title + thumbnail brain, Fact-check |
| `/sakana` | Agent | Sakana Fugu | "A vendor-agnostic council — a panel deliberates in parallel with web search, then a judge… writes the verdict." | Same boardroom UI |
| `/local` | Agent | Local | "Whatever model you've pinned warm in Ollama… offline, free, instant. Build with your voice, preview live." | Builder · "Maple · reasoning"; Build · Preview · Workspace |
| `/jcode` | Agent | jcode | "The most RAM-efficient harness — Rust, ~28MB a session, 14ms to first frame, on your Claude login." | Terminal · Workspace · History; cwd `~/.agentic-os/jcode` |
| `/agent-kanban` | Agent | Agent Kanban | "A team of local offline agents works a live board: Planner → Builder → Reviewer — every Done card previews live." | Board · Workspace; Backlog / Building / Review / Done |
| `/kanban` | Self | Kanban | "Hermes Agent multi-agent board. Drop a prompt into triage, watch the orchestrator decompose + assign." | Assignee filter · Dispatch now · Triage |
| `/herdr` | Orchestration | Herdr | "Persistent workspaces for your coding agents." | Interactive terminal: Reconnect · Disconnect · Full screen · key buttons · Split right/down · New tab |
| `/paperclip` | Self | Paperclip | "Clip anything into the vault … and let the agents file it." | Builds · Issues · Active · Done · Org · Dashboard · **Costs** |
| `/room` | Self | AI Agent Mastermind | "A live group chat with all your agents — each one a different real model… Tag one with @claude." | New chat · History · In the room |
| `/pipeline` | Self | From Inbox to Shipped | "Capture an idea → agents classify, route + plan it → you approve once → a PM + subagents build it. Lives in your vault." | Pipeline · Gallery; stages Capture → … |
| `/loop` | Self | Loop Engineering | "Define what 'done' looks like. A builder acts, Fusion verifies it adversarially, and it loops until the gate passes." | Check state → Decide → Act |
| `/memory` | Self | Memory | "Search 1261 Omi memories + your Obsidian vault." | Projects · Create · Recent · Notes · Omi · Graph · Galaxy ✦ |
| `/notebook` | Self | Notebook | "Your NotebookLM notebooks, audio overviews and chats — all in one place, synced to Obsidian." | Library · Shorts · Podcasts · Infographics · Decks · Reports · Studio · Assets · Research · Chat |
| `/radar` | Self | The Radar | "A 24/7 watcher on AI news + X. Hermes searches X live (Grok OAuth), ranks what's trending… Sweeps every morning, logs to Obsidian." | Consult the oracle · Hear the brief · Trending on X |
| `/apps` | Self | Apps | "Everything your agents have built, playable and previewable." | App Lab (from `~/Developer/awesome-llm-apps`, "Every app runs on OpenRouter free models") |
| `/opendesign` | Self | Open Design | "The local-first, open-source Claude Design alternative — embedded right here." | Start / Re-check (spawns a local service, "Node 24, no Docker") |
| `/seo` | Self | SEO Intelligence | "Search performance, keyword opportunities and your AI visibility." | SearchAgent OS · Research · Parasite SEO · OpenSEO · SEO Office · Generate · Deploy · History · Sites · Transcripts · Skill · Setup Guide · SEO Pack (.zip) |
| `/thumbnails` | Self | Thumbnail Studio | "Upload a thumbnail and describe your changes. Generate with Grok or OpenAI. Every round is logged to your vault so it learns your style." | Provider: Grok via Hermes / OpenAI |
| `/sales` | Workspace | Sales Agent OS | "Your team's conversations, coaching and daily history." | "Demo privacy is on"; Google Workspace reports; "daily sync runs in Codex" |
| `/ad-studio` | Growth | Ad Studio | "Generate ad creatives and copy from your existing ads." | Brief → reference creatives upload (3 × 5 MB) |
| `/ads` | Growth | Ads Performance | "Facebook and Instagram campaign reporting." | 7/14/30-day ranges, placements, export |
| `/design-os` | Create | Design OS | "Your ideas. Your design library. Beautiful websites." | Create · My websites · Design library · Website Generator |
| `/website-generator` | Create | Website Generator | "Build, preview and export local service websites." ("Rank & Rent") | Business / city brief → pages |
| `/worlds` | Create | World Studio | "Detailed Blender environments. Walk inside. Explore freely." | Embedded 3D viewer |
| `/video` | Studio | Video | "The Video Director: research → script → avatar + b-roll → edit." | HeyGen avatar, MiniMax / Grok b-roll, ElevenLabs |
| `/openmontage` | Studio | OpenMontage | "Agentic video editing — a real Remotion render from a prompt." | Veo movie / cinematic stills; length presets |
| `/video-use` | Studio | Video Use | "Point an agent at a video and have it watch, describe and act on what it sees." | Edit jobs · Upload · brief presets · Agent log |
| `/music` | Studio | Music | "Suno-powered tracks from a prompt." | Vibe presets, "Suno v5" |
| `/games` | Workflow | Game Studio | "Describe a game, it builds it, you play it on the shelf." | "Claude 5 brain · single-file games · no libraries" |
| `/hermes?tab=apollo` | — | Apollo | Voice agent (see §7.6) | — |

---

## 6. Backend: the Next.js API layer

- **Observed:** **283 distinct `/api/*` paths** are referenced across the bundles (full list in [Appendix A](#appendix-a--full-api-endpoint-inventory)). They are **Next.js Route Handlers** (`app/api/**/route.ts`) in the same app, on the same origin.
- **Namespacing convention:** one namespace per agent or feature, with a consistent sub-resource vocabulary:

| Sub-resource | Purpose |
|---|---|
| `chat` | POST a prompt, get a streamed response |
| `workspace` | list files the agent produced |
| `workspace/file` | read or write a single file |
| `workspace/raw` | raw bytes |
| `preview/<id>/<path>` | serve a generated file for iframe rendering |
| `builds`, `builds/file/<id>` | saved build artifacts |
| `history`, `sessions` | conversation persistence |
| `status`, `config`, `settings`, `model` | health and configuration |
| `studio/*` | media generation (image, video, TTS, search) |
| `scan`, `latest`, `publish`, `deploy` | background-job style endpoints |

- **Endpoint counts by namespace:** hermes 35, video 20, seo 15, openclaw 12, notebooklm 11, claude 10, codex 9, shorts / radar / pipeline / outreach / memory / freeclaude 7 each, antigravity 6, and about 45 smaller ones.
- **Implementation (inferred, from protocols and UI text):** the handlers do one of five things:
  1. **Spawn a CLI** with `child_process.spawn` and pipe its stdout as a stream. Examples: `claude`, `codex`, `kimi -p … --output-format stream-json`, `grok`, `opencode`, `hermes`, `jcode`, `antigravity`. Event types `stdout`, `stderr` and `exit` appear in the client parsers.
  2. **Call a local daemon** over HTTP: the OpenClaw gateway, Ollama (`localhost:11434`), the Herdr session server, the Open Design dev server, StarNet.
  3. **Call a cloud API** with keys held server-side: OpenAI (Realtime, image), OpenRouter, DeepSeek, Suno, HeyGen, ElevenLabs, Meta Ads, Google Workspace, Higgsfield (MCP).
  4. **Read or write the Obsidian vault** directly on the filesystem.
  5. **Serve files** from workspace folders under `~/.agentic-os/<agent>/…`.
- **Persistence is the filesystem.** There is no database client in the bundle (that is client-side only, so the server could still use SQLite), but everything visible maps to folders: vault paths like `Agentic OS/Pipeline/items/<slug>`, workspace dirs like `~/.agentic-os/jcode`, and "Builds and chat history survive a refresh" (saved as files).

---

## 7. Core integration patterns (the "how")

### 7.1 Agent adapter / "agent room" pattern

Every agent page is the same component tree with a different backend adapter:

```
<AgentPage agent="claude">
  <Header section="Agent · Claude" title desc />
  <Tabs> Chat | Workspace | Artifacts | … </Tabs>
  <ChatPane  endpoint="/api/claude/chat"  />      // streamed
  <Workspace endpoint="/api/claude/workspace" />  // file list + Monaco + preview iframe
  <Composer  voiceInput  autoSaveToObsidian />
</AgentPage>
```

The server side is effectively one interface implemented per agent:

```ts
interface AgentAdapter {
  id: string;                         // "claude" | "codex" | "openclaw" | …
  health(): Promise<Vitals>;          // feeds /api/vitals
  chat(input: { prompt: string; sessionId?: string; project?: string })
      : AsyncIterable<AgentEvent>;    // normalized events → NDJSON
  workspaceDir: string;               // ~/.agentic-os/<id>
}
```

**This is the single most reusable idea in the whole app:** new agents are cheap because adding one means one adapter plus one page from the template. That's why there are 30+ of them.

### 7.2 Chat streaming protocol (NDJSON over fetch)

- **Observed:** chat endpoints answer the `POST` with a **streaming response body of newline-delimited JSON**. The client reads it like this (de-minified):

```js
const res = await fetch("/api/room", { method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ message, history, agents }), signal });
const reader = res.body.getReader(), dec = new TextDecoder(); let buf = "";
for (;;) {
  const { value, done } = await reader.read(); if (done) break;
  const lines = (buf += dec.decode(value, { stream: true })).split("\n");
  buf = lines.pop() ?? "";
  for (const line of lines) if (line.trim()) handle(JSON.parse(line));
}
```

- **Event discriminators** seen across the parsers (`t`/`type` fields): `text`, `delta`, `done`, `error`, `typing`, `msg`, `agent`, `agent_message`, `reasoning`, `think`, `tool`, `tool_use`, `tool_result`, `step`, `step_end`, `stdout`, `stderr`, `exit`, `begin`, `end`, `result`, `system`, `assistant`, `user`, `stream_event`, `info`, `warn`, `stats`, `proposal`, `ideas`, `scan`, `ultracode_run_started` (carries `runId`).
- **Why NDJSON rather than SSE for chat:** it works with a `POST` body (EventSource is GET-only), it's trivial to produce from a Route Handler with a `ReadableStream`, and it passes through Cloudflare unbuffered.
- **Resumable runs:** Claude "Ultracode" posts `{prompt, ultracode:true, project}`. The stream emits `ultracode_run_started` with a `runId`, and the client can later re-attach with `{prompt, resumeRunId}`. History is managed via `GET/DELETE /api/claude/ultracode?id=`. So **long runs continue server-side after the browser disconnects.**

### 7.3 Claude Code integration (stream-json)

- **Observed:** the Claude client parser understands exactly the **Claude Code headless `stream-json` schema**:

```js
if (t.type === "stream_event" && t.event?.delta?.text) { /* token delta */ }
else if (t.type === "assistant" && t.message?.content) { /* full message blocks (type:"text") */ }
else if (t.type === "result") { /* final result + cost/usage */ }
```

- **Inferred:** the server runs roughly this and pipes stdout line by line to the browser:

  ```
  claude -p "<prompt>" --output-format stream-json --include-partial-messages --verbose [--resume <session>]
  ```

  `stream_event` with `delta.text` only appears with partial messages enabled.
- `/api/vitals` reports `"claude":{"ok":true,"version":"2.1.x (Claude Code)"}`, i.e. a `claude --version` probe.
- **Additional Claude endpoints:**
  - `/api/claude/ant`, `/ant/agents`, `/ant/agents/run`, `/ant/agents/trace`: an "Ant CLI" tab for running and tracing sub-agents.
  - `/api/claude/artifacts`, `/api/claude/workspace`, `/workspace/file`, `/preview/<project>/<relPath>`.
- **Kimi** uses the same approach: the UI literally states `kimi -p … --output-format stream-json`.
- **Auth model:** the CLIs run under the owner's **consumer subscriptions** ("on your Claude login", "Codex on your ChatGPT login · no API key", `/api/hermes/realtime/codex` → `"login":"chatgpt"`, Grok "signed in on your X Premium+ plan"). The server holds no API keys for these; it inherits the CLI's logged-in OAuth session on disk.

### 7.4 Workspace + live preview pattern

1. The agent is told, via a system prompt or project dir, to write output to `~/.agentic-os/<agent>/<project>/…`.
2. `GET /api/<agent>/workspace` lists files, and `/workspace/file?path=` reads one (shown in Monaco).
3. `GET /api/<agent>/preview/<project>/<relPath>` serves the file with the right content type, so relative asset links inside generated HTML resolve.
4. The UI renders it in an iframe: `<iframe src=… sandbox="allow-scripts allow-pointer-lock allow-same-origin">` (observed for the "Apollo voice build" preview).
5. "Save build" copies it into `builds/`, which appears on the Apps / Games shelf.

**Note:** `allow-scripts` + `allow-same-origin` on a **same-origin** preview route lets the previewed code reach the parent app and its `/api/*`. That's fine for trusted local toys and dangerous for anything untrusted (§10). A proper clone serves previews from a separate origin.

### 7.5 Web terminal (Herdr) over SSE + POST

- **Observed:**

```js
// 1. create a viewer
const { id } = await post("/api/herdr/terminal", { action: "connect", cols, rows });
// 2. downstream: server → browser
const es = new EventSource(`/api/herdr/terminal?id=${id}`);
es.onmessage = e => term.write(JSON.parse(e.data));
// 3. upstream: browser → server
term.onData(d   => post("/api/herdr/terminal", { action: "input",  data: d }));
term.onResize(s => post("/api/herdr/terminal", { action: "resize", cols: s.cols, rows: s.rows }));
// FitAddon + ResizeObserver keep the PTY sized to the pane
```

- **Herdr** is a persistent multi-pane terminal session manager (tmux-like): `~/.local/bin/herdr --session agent-os`. The UI help text says:
  - "Disconnect closes only this viewer. Your agent processes continue while the Mac is awake."
  - "A Mac restart restores layouts, not original processes."
  - Instructions to run `hermes`, `codex`, `claude` or `opencode` inside panes.
- **Why SSE + POST and not WebSockets:** Next.js Route Handlers don't natively support WebSocket upgrades. SSE streams fine from a Route Handler and through Cloudflare. Keystroke latency is one HTTP POST each, which is acceptable over HTTP/2 or HTTP/3.
- The same terminal component (different backends) powers `/jcode`, `/opencode`, `/prime` and `/glm-code`.

### 7.6 Voice agent "Apollo" (OpenAI Realtime over WebRTC)

**Observed flow:**

```
Browser                              Next.js server                OpenAI
  │ getUserMedia + RTCPeerConnection     │                            │
  │ createOffer() → SDP                  │                            │
  │─ POST /api/hermes/realtime/session ─▶│  mint ephemeral client key ─▶│
  │◀──────── { value, model:"gpt-realtime" } ◀────────────────────────│
  │─ POST https://api.openai.com/v1/realtime/calls?model=… ───────────▶│
  │      Authorization: Bearer <ephemeral>, Content-Type: application/sdp
  │◀──────────────────────────── SDP answer ───────────────────────────│
  │ setRemoteDescription(answer)   ⇄ audio over WebRTC, events on data channel
  │
  │ data channel: response.function_call_arguments.done
  │─ POST /api/hermes/realtime/open {target}  (open an app/page)
  │─ POST /api/hermes/realtime/see            (vision / "see" the screen)
  │─ GET  /api/hermes/realtime/codex          (Codex voice handoff)
  │ → send conversation.item.create {type:"function_call_output", call_id, output}
  │ → send response.create
  │─ POST /api/hermes/apollo-log, /apollo-memory   (transcript → vault)
```

- **Details observed:**
  - The SDP POST is retried up to 4 times, with specific handling for `429` + `insufficient_quota`.
  - User-facing error codes: `api_session`, `api_link` ("Couldn't open the realtime audio link."), `no_session`, `NotAllowedError` (mic permission).
  - The client keeps a session `id`, pings every 8 s (`{action:"ping",id}`), and stops with `fetch(..., {keepalive:true})` so teardown survives tab close.
- **Second mode:** a server-relayed path with `{action:"start", sdp}`, `{action:"text"}`, `{action:"speak"}`, `{action:"stop"}` and an `EventSource` delivering `{t:"delta"|"done"|"agent", role, delta|text}`.
  - It's tied to the **Codex** voice backend: `/api/hermes/realtime/codex` returns `{installed:true, login:"chatgpt", voices:["arbor","cove","ember","spruce","breeze","juniper","maple","sol","vale"], defaultVoice:"arbor"}`.
  - Session state includes `plan`, `idleMs`, `maxMs` and `ack`, which suggests idle and maximum session limits and an acknowledgement phrase.
- **Why it's good:** audio never transits the home PC, so latency stays low. The OpenAI key never reaches the browser (only a short-lived ephemeral key does). The voice model can **drive the rest of the OS** through tools.

### 7.7 Multi-agent room (Mastermind)

- `POST /api/room` `{message, history, agents:[…]}` returns NDJSON events:
  - `{t:"typing", id, name, color}`: shows "X is typing…"
  - `{t:"msg", …}`: appends a message
- Agents "reply in turn, riff off each other, and read your vault for context". `@mention` targets a single agent, and they "can save notes or start projects" (tool access to vault and pipeline).
- **Inferred server loop:** for each selected agent in order, build a prompt from the shared transcript plus vault context plus a persona, call that agent's adapter, stream the result as a `msg`, and append it to the transcript for the next agent.
- History lives at `/api/room/history`.

### 7.8 Councils / boardrooms (Fusion, Sakana, MoA)

- **Fusion** (`/api/fusion`, OpenRouter "Fusion") and **Sakana Fugu**:
  1. A **panel of models deliberates in parallel with web search**.
  2. A **judge model writes the verdict**.
  3. The UI warns "a full panel takes 30–90s".
  4. Presets: "SEO content council", "Title + thumbnail brain", "Fact-check".
- **Hermes "Mixture"** tab plus `/api/moa`, `/api/moa/workspace`, `/api/moa/file/<id>`: a **Mixture-of-Agents** endpoint (proposers, then an aggregator).

### 7.9 Loop engineering (builder + adversarial judge)

- `/loop`: "Define what *done* looks like." The cycle is **Check state → Decide → Act**:
  1. A builder agent acts.
  2. A "free judge (N2 by default — no per-token cost)" grades it **adversarially out of 100**.
  3. The loop repeats until the gate passes or progress stalls. "The builder never grades its own homework."
- **Prime Agent** applies the same idea in-harness: "bounded autonomous mode with quality gates… can't call the work done until the checks pass."

### 7.10 Pipeline (inbox → shipped)

- "Capture an idea → agents classify, route + plan it → **you approve once** → a PM + subagents build it."
- **Stages:** inbox → shaping → building → review → shipped.
- **Storage:** markdown in the vault under `Agentic OS/Pipeline/items/<slug>`.
- **Observed item schema** (from `GET /api/pipeline`): `slug`, `title`, `stage`, `created`, `updated`, `pinned`, `idea` and `vaultPath`.
- **Endpoints:** `/api/pipeline`, `/api/pipeline/capture`, plus 5 more.

### 7.11 Memory layer (Obsidian vault + Omi)

- **The vault is the system of record:** chats are auto-saved, and pipeline, radar, thumbnails ("logged to your vault so it learns your style"), NotebookLM ("synced to Obsidian") and Apollo transcripts all write there.
- **Endpoints:**
  - `/api/memory/search?q=`, `/memory/recent`, `/memory/note`, `/memory/create`, `/memory/projects`, `/memory/log`
  - `/api/memory/omi?limit=8`: memories from the **Omi** wearable (1,261 at analysis time)
- **Views:** Recent, Notes, Omi, **Graph**, **Galaxy ✦** (three.js particle rendering of the vault).
- **Inferred retrieval:** keyword or full-text search over markdown, then the top notes are injected into agent prompts. No vector-DB client was visible (it could still run server-side).

### 7.12 Health / vitals

`GET /api/vitals` (observed shape):

```json
{ "ts": 1793895224109,
  "claude":   { "ok": true, "version": "2.1.x (Claude Code)", "latencyMs": 37 },
  "openclaw": { "ok": true, "gateway": "live", "degraded": false, "busy": false,
                "loopMaxMs": 34, "loopP99Ms": 32, "agents": ["main"], "sessions": 1,
                "latencyMs": 1784, "raw": "Gateway event loop: ok … Heartbeat interval: 30m … Session store (main): C:\\Users\\…" },
  … }
```

The server shells out to each agent's status command (`claude --version`, `openclaw status`, …) and normalizes the result. It feeds the "All systems" pill and the "/5 agents online" counter.

### 7.13 Scheduled / background jobs

- **Radar:** "Sweeps every morning". Hermes searches X live via Grok OAuth and ranks signals.
  - Endpoints: `/api/radar/scan`, `/latest`, `/history`, `/draft`, `/publish`, `/published`, `/source`
  - `/latest` returns `rankingVersion`, `scannedAt`, `day` and `signals[{basis, headline, why_now, angle, …}]`.
- **Others:** Astros (`/api/astros/scan`, `/latest`, `/notebook`, `/config`, `/history`), Furnace (`/api/furnace/scan`, `/latest`), SEO (`/api/seo/deploy`, …), Shorts (`/api/shorts/run`, `/jobs`, `/post`, `/script`), Outreach (`/api/outreach/enrich`, `/validate`, `/write`, `/send`, `/inbox`), and a Sales "daily sync runs in Codex".
- **Inferred:** the scheduling itself is either Hermes/OpenClaw's built-in heartbeat (OpenClaw reports "Heartbeat interval: 30m") or OS-level cron / Task Scheduler hitting these endpoints. The Next app only exposes the endpoints and state.

---

## 8. Agent and model inventory

As claimed by the UI. Integration type is inferred from UI text and endpoints.

| Agent | Vendor / project | Integration | Auth model |
|---|---|---|---|
| Claude (Code) | Anthropic | spawn CLI, `stream-json` | Claude subscription login |
| Codex | OpenAI | spawn CLI | ChatGPT login (`"login":"chatgpt"`) |
| OpenClaw | open source | local gateway daemon | local; runs `ollama/deepseek-v4-flash` |
| Hermes | Nous Research | CLI / local service; skills, MCP, bots, kanban | Nous Portal; Grok OAuth for X search |
| OpenCode | open source | terminal (PTY) | any provider |
| Antigravity | Google ("Gemini CLI's successor") | spawn CLI | Google login |
| Kimi Code | Moonshot | `kimi -p … --output-format stream-json` | OAuth |
| GLM 5.2 / GLM Code | Zhipu z.ai | API / CLI | z.ai Coding Plan |
| Grok Build | xAI | CLI | X Premium+ login |
| Prime Agent | Prime Intellect | terminal harness | — |
| jcode | open source (Rust) | terminal | Claude subscription |
| Free Claude Code | open-source proxy | Claude CLI → local model | none |
| OmniRoute / 9Router / Ox Alpha / Step 5 | free gateways | HTTP gateway | none / own logins / Nous Portal |
| HY3, MiMo, Muse, DeepSeek | Tencent, Xiaomi, Meta, DeepSeek | OpenRouter / native API | API keys |
| Local | Ollama | HTTP `localhost:11434` | none |
| StarNet, OpenMausBot, Open Design | open source | embedded local apps | — |
| Fusion, Sakana Fugu | OpenRouter / Sakana | council API | API key |
| Media | OpenAI (gpt-image-2, Realtime), Suno, HeyGen, ElevenLabs, MiniMax, Grok, Veo, Remotion, Higgsfield (MCP), NotebookLM | APIs / MCP / local render | keys / logins |

**Takeaway:** the owner's design principle is **"use the subscriptions and CLIs you already pay for, free tiers and local models wherever possible"**. The app is a cost-aware router over consumer plans.

---

## 9. Visual design system

- **Themes:** at least three dark palettes, defined as CSS custom properties:
  - `--bg-deep`: `#15101a` (aubergine, default), `#111314` (graphite), `#151e19` (forest)
  - `--bg-mid`: `#1c1622` / `#17191b`
  - `--bg-card`: `#251d2c` / `#1b1e20` / `#151e19`
  - `--bg-elev`: `#2e2436` / `#24282b` / `#202d25`
- **Accent families:**
  - `--apx-*` / `--apb-*` "aurora" set: violet `#7c5cff`, cyan `#22d3ee`, mint `#2dd4bf`, pink `#ff4ecd`, blue `#60a5fa`, amber `#fbbf24` / `#ffb547`, rose `#fb7185`
  - Glass surfaces: `--apx-glass:#120e3080`, `--apb-glass:#100c2c8c`, hairlines `--apx-line:#a78bfa38`
- **Typography:** Bricolage Grotesque for display headings, Manrope for UI, JetBrains Mono for code and terminal, Caveat for playful handwritten hints.
- **Signature visuals:** a WebGL "core" orb (shader-displaced icosphere, additive particles), a glassmorphism card grid, per-agent accent colors (also used for the `typing` indicator colors in Mastermind), and a pixel-art space station (StarNet).
- **Terminal theme:** background `#101410`, foreground `#d6dfce`, cursor `#bfd2ad`, selection `#657b5555`.

---

## 10. Security assessment

> I didn't attempt any exploitation. Everything below comes from read-only `GET`s and static code analysis.

### 10.1 Findings

| # | Finding | Evidence | Severity |
|---|---|---|---|
| S1 | **No authentication on the API.** `GET /api/version`, `/api/vitals`, `/api/todos`, `/api/pipeline`, `/api/radar/latest`, `/api/seo/published` and `/api/hermes/realtime/codex` all returned data to an anonymous request from the internet. No login page, auth cookies, auth library or Cloudflare Access challenge was encountered. | Observed | High |
| S2 | **Remote shell / code-execution surface reachable over the same unauthenticated origin.** The client calls `POST /api/herdr/terminal` (`connect`/`input`/`resize`, a live PTY on the host), `POST /api/run`, and agent `chat` endpoints that spawn CLIs with file-system and shell tools. If these share S1's lack of auth, **anyone on the internet can run commands on the owner's PC**. *Not tested.* | Inferred | Critical (if confirmed) |
| S3 | **Outbound-action endpoints:** `/api/outreach/send`, `/api/seo/deploy`, `/api/radar/publish`, `/api/shorts/post`, `/api/hermes/mcp/install`, `/api/hermes/mcp/add`. Abuse could send email, publish content or install MCP servers under the owner's identity. | Observed (paths) | High |
| S4 | **Information disclosure:** `/api/vitals` leaks OS user-profile paths and internal agent topology; `/api/version` leaks the owner's name; `/api/todos` and `/api/pipeline` leak personal plans and ideas; `/api/memory/*` likely exposes the vault and Omi (wearable) memories. | Observed / inferred | High (privacy) |
| S5 | **Same-origin preview iframe with `allow-scripts allow-same-origin`.** Agent-generated (or prompt-injected) HTML runs with the app's own origin and can `fetch('/api/…')`. | Observed | Medium |
| S6 | **Cost and quota abuse:** anonymous callers could burn the owner's OpenAI Realtime quota (`/api/hermes/realtime/session` mints keys), OpenRouter credits, and the Claude, ChatGPT and X subscriptions. | Inferred | Medium |
| S7 | **Prompt injection via ingestion:** radar (X posts), Paperclip (arbitrary links/screenshots), outreach inbox and Omi memories all feed into agents that hold shell and publish tools. | Inferred | Medium |

### 10.2 What it should have

- **Cloudflare Access** (or oauth2-proxy / Entra ID) in front of the *whole* hostname, or Cloudflare Tunnel to a private network with WARP only.
- **Server-side auth** on every route handler as defense in depth, plus CSRF protection on POSTs.
- Previews on a **separate origin** (`preview.<domain>` or a `blob:`/sandboxed origin without `allow-same-origin`).
- A **tool and approval policy** for dangerous agent actions (shell, publish, send), and an allowlist for which agents may touch which tools.

> If you know the owner, let them know about S1/S2. Putting Cloudflare Access in front of the hostname is a ten-minute fix.

---

## 11. How to build your own

### 11.1 Minimal architecture (single-user, local-first, like the original but safe)

```
[Browser / iOS PWA]
     │ HTTPS (Cloudflare Access / Tailscale / Entra-protected ingress)
[Next.js app (App Router)] ── same host as the agents
     ├─ /app/(shell)        sidebar, ⌘K (cmdk), Mission Control, agent pages
     ├─ /app/api/<agent>/chat        → AgentAdapter.chat() → NDJSON stream
     ├─ /app/api/<agent>/workspace   → fs list/read under ~/.agentos/<agent>
     ├─ /app/api/<agent>/preview/*   → served from a SEPARATE origin
     ├─ /app/api/terminal            → node-pty / tmux, SSE down + POST up
     ├─ /app/api/voice/session       → mint OpenAI Realtime ephemeral key
     ├─ /app/api/memory/*            → markdown vault (+ optional vector index)
     └─ /app/api/vitals              → per-adapter health()
Agents: claude (Agent SDK or CLI stream-json), codex, opencode, ollama, …
Vault:  ~/vault (Obsidian-compatible markdown, git-backed)
```

### 11.2 Build order (each step is a self-contained vibe-coding prompt)

1. **Shell:** Next.js + Tailwind + cmdk + vaul + lucide; sidebar from a config array; dark theme tokens; PWA metas.
2. **Adapter interface + Claude adapter:** use the **Claude Agent SDK** (TypeScript) rather than parsing CLI stdout. It gives you typed messages, sessions/resume, tool permission callbacks, hooks and MCP. Fall back to `claude -p --output-format stream-json --include-partial-messages` if you want subscription auth.
3. **NDJSON chat UI:** a generic `<ChatPane endpoint>` that renders `text` / `tool_use` / `tool_result` / `result` events, with session resume.
4. **Workspace + preview:** file tree + Monaco + iframe preview on a second origin.
5. **More adapters:** Codex CLI, OpenCode, Ollama (HTTP), OpenRouter (HTTP). Each is about 100 lines.
6. **Vault memory:** write every turn as markdown with front-matter; simple full-text search first, embeddings later; inject top-k notes into system prompts.
7. **Vitals:** a health probe per adapter and an "All systems" pill.
8. **Terminal:** xterm.js + `@xterm/addon-fit` + node-pty (or `tmux` control mode) over SSE + POST. Use WebSockets if you run a custom server instead of Route Handlers.
9. **Voice:** OpenAI Realtime WebRTC with server-minted ephemeral keys and 3–5 tools (`open_page`, `ask_agent`, `capture_idea`, `search_memory`).
10. **Orchestration toys:** Mastermind room (sequential fan-out), council (parallel proposers + judge), loop (builder + judge gate), kanban (planner / builder / reviewer).
11. **Jobs:** a scheduler (cron, k8s CronJob, or the agent's own heartbeat) calling job endpoints; results land in the vault.

### 11.3 Prompting strategy that reproduces it fastest

- Write a `CLAUDE.md` with: the adapter interface, the NDJSON event schema, folder conventions (`~/.agentos/<agent>`, vault layout), the theme tokens, and "every new agent = one adapter + one page from the template".
- Then build feature by feature: *"Add an agent page for OpenCode using the AgentAdapter template, with a terminal tab and a workspace tab."* That incremental "add X to my Agent OS" loop is how the original grew to 50 pages.

### 11.4 Things to do better than the original

- Auth first (see §10.2).
- One normalized event schema across adapters (the original has many slightly different ones).
- A real job runner with retries and logs instead of ad hoc scan endpoints.
- A cost and usage ledger per adapter (the Paperclip "Costs" tab hints at this).
- Git-back the vault so agent writes are diffable and revertible.

---

## 12. Platform-grade variant (k8s / AKS / OpenShift / k3s)

The original is single-user and runs as the owner on their desktop. Hosting it on a cluster changes the hard problems:

| Concern | Desktop original | Cluster-grade approach |
|---|---|---|
| Agent execution | `spawn()` on the host, full user privileges | **One sandboxed pod per session** (gVisor `runsc` / Kata RuntimeClass on AKS or OpenShift sandboxed containers); a controller or operator creates and reaps them; network egress via allowlisted proxy |
| Agent auth | CLI OAuth logins on disk | Per-tenant API keys (Anthropic / Azure OpenAI / Foundry) from **Key Vault via CSI driver** or External Secrets; Workload Identity; no consumer-plan tokens in shared infra |
| State | Obsidian vault + `~/.agentic-os` | Vault = **git repo per user/team** (agents commit); workspaces = PVC per session or object storage; optional Postgres + pgvector for search |
| Streaming | NDJSON / SSE through Cloudflare | Same protocols; ingress tuned for long-lived streams (timeouts, buffering off); WebSockets fine behind NGINX / Envoy / OpenShift Routes |
| Terminal | Herdr on the host | `kubectl exec`-style attach into the session pod via a broker (never a host shell) |
| Auth | none | Entra ID OIDC at ingress (oauth2-proxy / Azure Front Door / Gateway API + ext-auth), plus per-route authz in the app |
| Previews | same-origin iframe | Separate wildcard preview domain, served from the session pod via an authenticated proxy |
| Jobs | morning sweeps | **CronJobs** / KEDA-scaled workers calling the same adapters |
| Delivery | `next start` on a PC | Container image, GitOps (Argo CD / Flux), Terraform for AKS / Key Vault / DNS, policy (Kyverno / Gatekeeper) for sandbox RuntimeClass enforcement |
| Observability | `/api/vitals` | OpenTelemetry traces per agent run, token and cost metrics, audit log of tool calls |

---

## 13. Open questions before a detailed design

1. **Audience:** just you (workstation / homelab k3s) or your team (AKS / OpenShift)?
2. **Execution scope:** should agents run code and terminals against your infrastructure, or only chat and build previews?
3. **Model providers and compliance:** direct Anthropic / OpenAI APIs, Azure OpenAI / Foundry only, local models, data residency?
4. **Memory:** Obsidian-compatible markdown in git, or a team knowledge store?
5. **Must-have features for v1:** terminal, voice, multi-agent room, pipeline/kanban, content tools?

---

## Appendix A — full API endpoint inventory

All 283 `/api/*` paths referenced in the client bundles, grouped by namespace (count in parentheses). Templated segments such as `${id}` are truncated to their static prefix.

**`hermes`** (35): `/api/hermes/apollo`, `/api/hermes/apollo-log`, `/api/hermes/apollo-memory`, `/api/hermes/bots`, `/api/hermes/briefing`, `/api/hermes/chat`, `/api/hermes/dashboard`, `/api/hermes/goals`, `/api/hermes/goals/files`, `/api/hermes/goals/preview`, `/api/hermes/kanban/action`, `/api/hermes/kanban/board`, `/api/hermes/kanban/dispatch`, `/api/hermes/kanban/task`, `/api/hermes/kanban/workspace`, `/api/hermes/kanban/workspace/file`, `/api/hermes/kanban/workspace/raw`, `/api/hermes/mcp`, `/api/hermes/mcp/add`, `/api/hermes/mcp/install`, `/api/hermes/mcp/manifest`, `/api/hermes/mcp/tools`, `/api/hermes/preview`, `/api/hermes/preview/glm`, `/api/hermes/profiles`, `/api/hermes/realtime/codex`, `/api/hermes/realtime/open`, `/api/hermes/realtime/see`, `/api/hermes/realtime/session`, `/api/hermes/studio/generate`, `/api/hermes/studio/list`, `/api/hermes/studio/video-status`, `/api/hermes/tts`, `/api/hermes/workspace`, `/api/hermes/workspace/file`

**`video`** (20): `/api/video/auto/assemble`, `/api/video/auto/example`, `/api/video/auto/script`, `/api/video/elevenlabs`, `/api/video/elevenlabs/connection`, `/api/video/elevenlabs/mcp`, `/api/video/elevenlabs/media`, `/api/video/heygen/avatars`, `/api/video/heygen/generate`, `/api/video/heygen/status`, `/api/video/heygen/voices`, `/api/video/hyperframes/init`, `/api/video/hyperframes/keyframes`, `/api/video/hyperframes/projects`, `/api/video/hyperframes/render`, `/api/video/hyperframes/render/status`, `/api/video/motion`, `/api/video/preview/project`, `/api/video/voices`, `/api/video/workspace`

**`seo`** (15): `/api/seo/deploy`, `/api/seo/generate`, `/api/seo/google-ai`, `/api/seo/history`, `/api/seo/intelligence`, `/api/seo/parasite`, `/api/seo/published`, `/api/seo/research`, `/api/seo/site-posts`, `/api/seo/sites`, `/api/seo/skill`, `/api/seo/skills`, `/api/seo/transcript/save`, `/api/seo/transcripts`, `/api/seo/visibility`

**`openclaw`** (12): `/api/openclaw/chat`, `/api/openclaw/preview`, `/api/openclaw/studio/chat-quick`, `/api/openclaw/studio/image`, `/api/openclaw/studio/list`, `/api/openclaw/studio/searches`, `/api/openclaw/studio/talks`, `/api/openclaw/studio/tts`, `/api/openclaw/studio/video`, `/api/openclaw/studio/xsearch`, `/api/openclaw/workspace`, `/api/openclaw/workspace/file`

**`notebooklm`** (11): `/api/notebooklm/artifact/download`, `/api/notebooklm/ask`, `/api/notebooklm/health`, `/api/notebooklm/library`, `/api/notebooklm/notebooks`, `/api/notebooklm/research`, `/api/notebooklm/research/import`, `/api/notebooklm/shortvideo`, `/api/notebooklm/shortvideo/recent`, `/api/notebooklm/shortvideo/status`, `/api/notebooklm/studio`

**`claude`** (10): `/api/claude/ant`, `/api/claude/ant/agents`, `/api/claude/ant/agents/run`, `/api/claude/ant/agents/trace`, `/api/claude/artifacts`, `/api/claude/chat`, `/api/claude/preview`, `/api/claude/ultracode`, `/api/claude/workspace`, `/api/claude/workspace/file`

**`codex`** (9): `/api/codex/chat`, `/api/codex/chats`, `/api/codex/goals`, `/api/codex/preview`, `/api/codex/session`, `/api/codex/session-file`, `/api/codex/sessions`, `/api/codex/workspace`, `/api/codex/workspace/file`

**`freeclaude`** (7): `/api/freeclaude/build`, `/api/freeclaude/builds`, `/api/freeclaude/chat`, `/api/freeclaude/preview`, `/api/freeclaude/preview/free-claude-code`, `/api/freeclaude/workspace`, `/api/freeclaude/workspace/file`

**`memory`** (7): `/api/memory/create`, `/api/memory/log`, `/api/memory/note`, `/api/memory/omi`, `/api/memory/projects`, `/api/memory/recent`, `/api/memory/search`

**`outreach`** (7): `/api/outreach`, `/api/outreach/enrich`, `/api/outreach/inbox`, `/api/outreach/send`, `/api/outreach/settings`, `/api/outreach/validate`, `/api/outreach/write`

**`pipeline`** (7): `/api/pipeline`, `/api/pipeline/build`, `/api/pipeline/capture`, `/api/pipeline/decide`, `/api/pipeline/delete`, `/api/pipeline/pin`, `/api/pipeline/shape`

**`radar`** (7): `/api/radar/draft`, `/api/radar/history`, `/api/radar/latest`, `/api/radar/publish`, `/api/radar/published`, `/api/radar/scan`, `/api/radar/source`

**`shorts`** (7): `/api/shorts/config`, `/api/shorts/file`, `/api/shorts/jobs`, `/api/shorts/post`, `/api/shorts/posts`, `/api/shorts/run`, `/api/shorts/script`

**`antigravity`** (6): `/api/antigravity/chat`, `/api/antigravity/models`, `/api/antigravity/preview`, `/api/antigravity/workspace`, `/api/antigravity/workspace/file`, `/api/antigravity/workspace/raw`

**`agent-kanban`** (5): `/api/agent-kanban/build`, `/api/agent-kanban/plan`, `/api/agent-kanban/preview`, `/api/agent-kanban/state`, `/api/agent-kanban/workspace`

**`astros`** (5): `/api/astros/config`, `/api/astros/history`, `/api/astros/latest`, `/api/astros/notebook`, `/api/astros/scan`

**`glm-code`** (5): `/api/glm-code/build`, `/api/glm-code/history`, `/api/glm-code/preview`, `/api/glm-code/status`, `/api/glm-code/workspace`

**`opencode`** (5): `/api/opencode/build`, `/api/opencode/history`, `/api/opencode/preview`, `/api/opencode/status`, `/api/opencode/workspace`

**`dsh`** (4): `/api/dsh/plugins`, `/api/dsh/run`, `/api/dsh/serve`, `/api/dsh/status`

**`higgs`** (4): `/api/higgs/gallery`, `/api/higgs/history`, `/api/higgs/run`, `/api/higgs/status`

**`jcode`** (4): `/api/jcode/build`, `/api/jcode/preview`, `/api/jcode/sessions`, `/api/jcode/workspace`

**`kimi`** (4): `/api/kimi/chat`, `/api/kimi/preview`, `/api/kimi/workspace`, `/api/kimi/workspace/file`

**`local`** (4): `/api/local/builds`, `/api/local/builds/file`, `/api/local/chat`, `/api/local/model`

**`music`** (4): `/api/music/generate`, `/api/music/list`, `/api/music/save`, `/api/music/status`

**`opendesign`** (4): `/api/opendesign/control`, `/api/opendesign/preview`, `/api/opendesign/projects`, `/api/opendesign/status`

**`prime`** (4): `/api/prime/build`, `/api/prime/preview`, `/api/prime/sessions`, `/api/prime/workspace`

**`thumbnails`** (4): `/api/thumbnails/file`, `/api/thumbnails/generate`, `/api/thumbnails/history`, `/api/thumbnails/research`

**`videouse`** (4): `/api/videouse/file`, `/api/videouse/jobs`, `/api/videouse/run`, `/api/videouse/status`

**`appslab`** (3): `/api/appslab/catalog`, `/api/appslab/run`, `/api/appslab/stop`

**`design-os`** (3): `/api/design-os`, `/api/design-os/preview`, `/api/design-os/worlds/assets`

**`dscoder`** (3): `/api/dscoder/chat`, `/api/dscoder/status`, `/api/dscoder/workspace`

**`hy3coder`** (3): `/api/hy3coder/chat`, `/api/hy3coder/status`, `/api/hy3coder/workspace`

**`loop`** (3): `/api/loop/builds`, `/api/loop/nous-models`, `/api/loop/run`

**`mimocoder`** (3): `/api/mimocoder/chat`, `/api/mimocoder/status`, `/api/mimocoder/workspace`

**`moa`** (3): `/api/moa`, `/api/moa/file`, `/api/moa/workspace`

**`musecoder`** (3): `/api/musecoder/chat`, `/api/musecoder/status`, `/api/musecoder/workspace`

**`omniroute`** (3): `/api/omniroute/chat`, `/api/omniroute/status`, `/api/omniroute/workspace`

**`openmausbot`** (3): `/api/openmausbot`, `/api/openmausbot/preview`, `/api/openmausbot/workspace`

**`content`** (2): `/api/content/board`, `/api/content/preview`

**`furnace`** (2): `/api/furnace/latest`, `/api/furnace/scan`

**`fusion`** (2): `/api/fusion/chat`, `/api/fusion/history`

**`herdr`** (2): `/api/herdr`, `/api/herdr/terminal`

**`openmontage`** (2): `/api/openmontage/generate`, `/api/openmontage/status`

**`paperclip`** (2): `/api/paperclip/builds`, `/api/paperclip/config`

**`room`** (2): `/api/room`, `/api/room/history`

**`sakana`** (2): `/api/sakana/chat`, `/api/sakana/history`

**`ad-studio`** (1): `/api/ad-studio`

**`ads`** (1): `/api/ads`

**`fcc`** (1): `/api/fcc`

**`games`** (1): `/api/games/commission`

**`glm`** (1): `/api/glm/chat`

**`grok`** (1): `/api/grok/chat`

**`openseo`** (1): `/api/openseo/status`

**`run`** (1): `/api/run`

**`sales`** (1): `/api/sales`

**`seooffice`** (1): `/api/seooffice/status`

**`starnet`** (1): `/api/starnet/status`

**`todos`** (1): `/api/todos`

**`uploads`** (1): `/api/uploads`

**`version`** (1): `/api/version`

**`vitals`** (1): `/api/vitals`

**`website-generator`** (1): `/api/website-generator`


---

## Appendix B — raw evidence snippets

Minified code excerpts lightly reformatted for readability.

**B.1 Realtime voice: SDP exchange with retry**
```js
const url = `https://api.openai.com/v1/realtime/calls?model=${encodeURIComponent(model)}`;
const headers = { Authorization: `Bearer ${ephemeralKey}`, "Content-Type": "application/sdp" };
for (let i = 0; i < 4; i++) {
  const r = await fetch(url, { method: "POST", body: offerSdp, headers });
  if (r.ok) return await r.text();               // SDP answer
  const body = await r.text().catch(() => "");
  if (r.status === 429 && /insufficient_quo/.test(body)) { /* quota error */ }
  await new Promise(res => setTimeout(res, 1000));
}
throw new VoiceError("api_link", "Couldn't open the realtime audio link.");
```

**B.2 Realtime voice: session bootstrap**
```js
const s = await fetch("/api/hermes/realtime/session",
  { method: "POST", headers, body: JSON.stringify({ voice }) }).then(r => r.json());
if (!s?.value) throw new VoiceError("api_session", s?.error || "Couldn't start a Realtime API session.");
const answer = await sdpExchange(s.model || "gpt-realtime", s.value, pc.localDescription.sdp);
await pc.setRemoteDescription({ type: "answer", sdp: answer });
```

**B.3 Realtime voice: tool-call round trip**
```js
// on data-channel event "response.function_call_arguments.done":
const r = await fetch("/api/hermes/realtime/open",
  { method: "POST", headers, body: JSON.stringify({ target }) }).then(r => r.json());
send({ type: "conversation.item.create",
       item: { type: "function_call_output", call_id, output: JSON.stringify(result) } });
send({ type: "response.create" });
```

**B.4 Relay-mode voice transcript over SSE**
```js
const es = new EventSource(`${relayUrl}?id=${encodeURIComponent(id)}`);
es.onmessage = e => {
  const m = JSON.parse(e.data); const role = m.role === "user" ? "user" : "assistant";
  if (m.t === "delta" && m.delta) onDelta(role, m.delta);
  else if (m.t === "done" && typeof m.text === "string") onFinal(role, m.text.trim());
  else if (m.t === "agent") setState("working");
};
// keepalive: POST {action:"ping",id} every 8s; stop: POST {action:"stop",id} with keepalive:true
```

**B.5 Claude Code stream-json parsing**
```js
if (t.type === "stream_event" && t.event?.delta?.text) { streamed = true; acc += t.event.delta.text; render(acc); }
else if (t.type === "assistant" && t.message?.content && !streamed) {
  acc = t.message.content.filter(b => b.type === "text").map(b => b.text).join(""); render(acc);
} else if (t.type === "result") { /* finalize */ }
```

**B.6 Ultracode resumable run**
```js
const res = await fetch("/api/claude/chat", { method: "POST", headers: { "content-type": "application/json" },
  body: JSON.stringify(resumeRunId ? { prompt, resumeRunId } : { prompt, ultracode: true, project }) });
// scans the stream for /"ultracode_run_started"[^}]*"runId":"([^"]+)"/ and stores runId
// DELETE /api/claude/ultracode?id=<runId> removes a run from history
```

**B.7 Herdr terminal**
```js
async function term(body) {
  const r = await fetch("/api/herdr/terminal", { method: "POST",
    headers: { "Content-Type": "application/json" }, body: JSON.stringify(body),
    signal: AbortSignal.timeout(10_000) });
  const j = await r.json(); if (!r.ok) throw Error(j.error || "Terminal request failed."); return j;
}
// theme: { background:"#101410", foreground:"#d6dfce", cursor:"#bfd2ad", selectionBackground:"#657b5555" }
const fit = new FitAddon(); xterm.loadAddon(fit); xterm.open(el); fit.fit();
const { id } = await term({ action: "connect", cols: Math.max(20, xterm.cols), rows: Math.max(5, xterm.rows) });
const es = new EventSource(`/api/herdr/terminal?id=${encodeURIComponent(id)}`);
es.onmessage = e => xterm.write(JSON.parse(e.data));
es.onerror = () => warn("Terminal connection closed. Your Herdr workspaces keep running.");
xterm.onData(d => term({ action: "input", data: d }));
xterm.onResize(e => term({ action: "resize", cols: Math.max(20, e.cols), rows: Math.max(5, e.rows) }));
new ResizeObserver(() => fit.fit()).observe(el);
```

**B.8 Mastermind room NDJSON**
```js
fetch("/api/room", { method: "POST", headers: { "content-type": "application/json" },
  body: JSON.stringify({ message, history, agents: [...selected] }), signal });
// per line: {t:"typing", id, name, color} | {t:"msg", ...}
```

**B.9 Preview iframe**
```jsx
<iframe src={previewUrl(file)} title="Apollo voice build"
        sandbox="allow-scripts allow-pointer-lock allow-same-origin" />
```

**B.10 three.js core**
```js
const renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true, powerPreference: "low-power" });
const core = new THREE.Mesh(new THREE.IcosahedronGeometry(1, 64),
  new THREE.ShaderMaterial({ uniforms, vertexShader, fragmentShader, transparent: true }));
const sparks = new THREE.Points(geo, new THREE.ShaderMaterial({ transparent: true, depthWrite: false,
  blending: THREE.AdditiveBlending,
  fragmentShader: "… float d = length(gl_PointCoord-.5); gl_FragColor = vec4(vC, smoothstep(.5,0.,d)*vA); }" }));
```

**B.11 Public API response shapes (personal values redacted)**
```jsonc
// GET /api/version
{ "version": "2026-10-18", "userName": "<redacted>" }
// GET /api/hermes/realtime/codex
{ "ok": true, "installed": true, "login": "chatgpt", "active": 0,
  "voices": ["arbor","cove","ember","spruce","breeze","juniper","maple","sol","vale"], "defaultVoice": "arbor" }
// GET /api/pipeline  (item shape)
{ "available": true, "items": [{ "slug": "…", "title": "…", "stage": "inbox", "created": "…", "updated": "…",
  "pinned": false, "idea": "…", "vaultPath": "Agentic OS/Pipeline/items/…" }] }
// GET /api/todos
{ "date": "YYYY-MM-DD", "today": "YYYY-MM-DD", "todos": [], "days": [{ "date": "…", "total": 1, "done": 0 }] }
// GET /api/radar/latest
{ "ok": true, "rankingVersion": 3, "scannedAt": "…", "day": "…",
  "signals": [{ "basis": "discussion", "headline": "…", "why_now": "…", "angle": "…" }] }
// GET /api/seo/published
{ "items": [], "sites": [{ "id": "…", "name": "…", "url": "…", "total": 0, "week": 0 }] }
```

---

## Sources

- Target site: https://agentos.maddogmurdockai.com/ (HTML, JS/CSS bundles, public GET endpoints; retrieved 2026-11-05)
- [Agent OS: Run Claude, Hermes & OpenClaw Together From One Dashboard (openclawdatabase)](https://openclawdatabase.com/news/videos/2026-05-16-agent-os-claude-hermes-openclaw-dashboard/index.md)
- [Same video summary, mirror (opcldatabase.pages.dev)](https://opcldatabase.pages.dev/news/videos/2026-05-16-agent-os-claude-hermes-openclaw-dashboard/)
- [Hermes Agent OS: Run All Your AI Agents from One Dashboard](https://openclawdatabase.com/news/videos/2026-06-01-hermes-agent-os-unified-ai-dashboard/index.md)
- [Claude as AI OS: Build a Command Center with Shared Memory](https://openclawdatabase.com/news/videos/2026-06-08-claude-agent-os-command-center/index.md)
- [Hermes "agent OS": brain, mission control and memory-vault layers](https://openclawdatabase.com/news/videos/2026-07-31-hermes-agent-os-three-layers/index.md)
- [Julian Goldie Podcast — AI News Today](https://share.transistor.fm/s/1372ddc1)
- Unrelated projects sharing the "AgentOS" name (for disambiguation): [Rivet agentOS](https://rivet.dev/agentos/docs/), [framersai/agentos](https://awesome.ecosyste.ms/projects/github.com%2Fframersai%2Fagentos), [AskUI AgentOS](https://docs.askui.com/06-agent-os), [AG2](https://app.ag2.ai), [Agent OS (Builder Methods)](https://github.com/JoshuaEstes/agent-os)
