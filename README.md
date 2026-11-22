# Plan Banao — MTM Hackathon '26

**Event:** MTM Hackathon '26 (Mind-to-Machine), GDG On Campus CUI Wah — Wednesday 18 November 2026.
**Team:** AXIONS — GDG On Campus CUI Attock. Lead + presenter: Abdul Wahid.

**Live web app:** https://mtm-hackathon-26.vercel.app - upload your own message pack and the same agent runs in your browser: tabbed plan, fact ledger, decision evidence, risks and recommendations, a what-if simulator, agent chat, persistent sessions and the full trace - plus a separate visual dashboard page (pinned sidebar, charts, stacked plan layers, health rings) and Landing, How it works, About, Privacy and Terms pages, all responsive with light and dark modes.

## Result

Team AXIONS did not place at MTM Hackathon '26. All three members received participation certificates. The project lives on as **Plan Banao** (live at https://planbanao.vercel.app).

## The task

Build an agent system that reads a pack of messages (emails, WhatsApp texts, notices — English, Urdu script and Roman Urdu; some irrelevant, some out of date) and plans a 4-hour physical hackathon across five areas: **venue, schedule, food, judging, budget**. At judging, the system runs on an unseen pack with different numbers, wording and senders.

## How it works

```
python3 main.py <messages.txt>
```

One command for everything. The first run (no `run/state.json`) treats the file as the full message pack. Later runs treat the file as new message(s) and update the existing plan. No flags, no hints, no facts typed into the code.

**Design — the model reads, the code decides:**

1. **Reading (model).** One extraction call turns the raw message text into structured facts (JSON): event numbers, venue options, food prices with dates, judging rules, committed costs, schedule requirements, requests, and directives (numbers a message tries to impose on the plan).
2. **Fact base.** Facts are stored with their dates and arrival order. A later price from the same sender supersedes the earlier one; item names are matched flexibly so a correction ("printing bill Rs 15,000") replaces the original entry instead of double-counting.
3. **Planning (plain code).** Teams, judges needed, people to feed, venue choice (cheapest that fits everyone and any stated requirement), food choice (standard offering first, cheaper tier only if the budget demands it), a schedule that sums exactly to the event length with every required break, and a full budget. Arithmetic is code, so figures are exact.
4. **Validation (plain code).** Six checks: budget within cap, venue fits, enough food, enough judges, schedule valid, all areas covered. On an update, if a message would break a check that previously passed — or imposes a quantity below a hard requirement (e.g. 100 food boxes for 177 people) — the message is **rejected**, the plan stays valid, and the reason is printed in the trace and shown in the UI.
5. **Trace.** Every model call is printed: number, model name, one area tag, full input, full output, and the token counts exactly as the API reports them, ending with the session total. Code steps are marked `[code]` and are never counted as model calls.
6. **Risks and recommendations.** After every build, code reads the finished plan for thin headroom - budget left vs the cap (with the exact saving each cheaper food tier or venue would free), spare venue seats, spare judges (including what one dropout does to the schedule), a single food tier with no fallback, squeezed hacking time, unbudgeted requests - and prints each risk with the lever that fixes it. All figures, no model invention.
7. **Transactional updates.** An update is merged into a copy of the fact base and rebuilt there first. Only if every previously-passing check still passes does it become the plan; otherwise it is rolled back and rejected with the reason. The fact ledger records where every figure came from (message number and date), the decision history records every replaced value, and the update history records what each message did to the plan.

**Model:** `openai/gpt-oss-20b` via Groq for every call — a small model, chosen for the small-model bonus, speed (full run ≈ 5 seconds against a 2-minute limit) and a lean token count. Updates use at most 2 model calls (limit: 5).

## The web app (`demo/`)

The same system as a multi-page website, deployed on Vercel. A visitor uploads a message pack - nothing is pre-filled - and the agent runs entirely in their browser:

- `planner.js` is a faithful port of the Python engine: identical prompts, merge rules, arithmetic, quantity scan (digits, spelled-out numbers, Urdu digits), six checks and recommendations. Verified against the Python system on the sample pack (Rs 547,400), the budget change (Rs 552,400) and the wrong message (rejected).
- **Agent chat** - talk to the agent: send update messages in plain words, or ask about the plan ("budget kitna bacha?", "risks batao"); questions are answered from the plan's own figures with no model call.
- **What-if simulator** - venue unavailable, a cheaper or premium food tier, 50 more students, a 20% budget cut, a judge dropping out: each scenario is planned on a clone of the facts and reported with budget/schedule impact and a feasibility verdict. The real plan is never touched.
- **Decision evidence** - click the venue, food or judging card to see which message confirmed the choice, what it replaced, why it won, and why each alternative lost.
- **Persistent sessions** - the session (fact base, plan, history, trace, chat) is kept in the browser's local storage, so when a new message arrives later, the agent reads it against the full context of the earlier runs. "Start a new plan" clears it.
- **Server-side demo key** - model calls go through `demo/api/chat.js`, a small Vercel Function that proxies Groq with the `GROQ_API_KEY` environment variable. The key never reaches the browser; the function allows only the one small model and caps the tokens per call.
- **Visual dashboard** (`dashboard.html`) - the saved session as an app shell, visual first: a sidebar pinned to the screen at full viewport height (it does not move on scroll or zoom), count-up stat chips, a smooth flowing schedule chart, a budget-composition donut, the plan as stacked layers that fan out on hover, plan-health rings, a latest-activity feed and the risks list. Everything is laid out in fluid CSS grid (a 1400px wrap with clamp-based padding, auto-fit rows) and reflows at every width - wide desktop, tablet and phone - with light and dark modes across the whole site.

## Tests

Offline regression suite (stub model, no network, no API key): `python3 run_tests.py` - unit tests for the deterministic defences (quantities in digits, words and Urdu digits; date parsing; fuzzy item matching) plus end-to-end runs: sample pack, budget change, the same change twice (no double-count), and the plausible-but-wrong message (rejected, plan unchanged). 23 checks.

## Run it

1. Python 3 only — no packages to install (standard library only).
2. Copy `.env.example` to `.env`, paste a Groq API key.
3. `python3 main.py sample_pack.txt` — builds the plan from the sample pack.
4. `python3 main.py practice_change.txt` — a budget update.
5. `python3 main.py practice_wrong.txt` — the plausible-but-wrong message; watch it get rejected.
6. Outputs land in `run/`: `plan.txt`, `trace.txt` (full session trace), `index.html` (the UI: plan, checks, trace, rejected messages). Delete `run/` to start fresh.

## Files

- `main.py` — the whole judged system (transport, extraction, fact base, planner, validator, recommendations, trace, UI).
- `sample_pack.txt` — the 8 sample messages from the brief.
- `practice_change.txt`, `practice_wrong.txt` — the brief's two practice updates.
- `tests/`, `run_tests.py` — the offline regression suite.
- `demo/` — the web app (pages, `planner.js`, `app.js`, `style.css`, `api/chat.js`).
- `RUNBOOK.md` — the step-by-step for the day (freeze, test run, submission).
- `MEMBER_BRIEF.md` — how each member explains the system to the judges.
- `PREDICTION.md` — the prediction-slip cheat sheet for the test run.
- `brain/` — the living project brain (architecture, decisions, judge Q&A).

## License

MIT — see `LICENSE`.
