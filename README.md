# AI Recruiter Copilot

**India Runs Hackathon · Track 1 — Data & AI Challenge: Intelligent Candidate Discovery**

A multi-agent system that reads job descriptions and candidate profiles the
way they actually show up in non-metro India — informal text, voice-note
style language, mixed Hindi/English — and returns a ranked, explainable
shortlist. No keyword matching: the agents reason about real skills, even
when those skills are never stated as keywords.

> "Worked at my uncle's hardware shop, handled customers and cash counter
> daily" should score *higher* for a sales role than a resume that just says
> "Sales Associate" with no detail — because the first one is evidence, and
> the second one is a label. That's the whole thesis of this project.

---

## What's in this repo

| Path | What it is |
|---|---|
| `index.html` (or `recruiter_copilot_prototype.html`) | **The working demo.** Self-contained HTML/JS that calls Claude directly from the browser. Open it inside a Claude.ai artifact and it just works. |
| `backend/agents.py` | The same three agents (JD Parser, Candidate Retriever, Evaluator+Ranker+Explainer), written as a deployable Python pipeline. |
| `backend/main.py` | FastAPI server exposing the pipeline as a `POST /rank` endpoint, for standalone deployment. |
| `backend/sample_data.json` | Sample JD + 6 candidate profiles for testing. |

---

## Architecture

```
 Input Layer                Multi-Agent Layer                Data Layer            Output
┌──────────────┐      ┌─────────────────────────┐      ┌──────────────────┐   ┌─────────────────┐
│ JD (text/     │      │  JD Parser Agent         │      │ Multilingual      │   │ Ranked Shortlist │
│ voice, any    │ ───► │  Candidate Evaluator     │ ───► │ Embedding Store    │──►│ + Explanations    │
│ language)     │      │  Ranking Agent           │      │ Candidate Profile  │   │ → WhatsApp Bot /  │
│ Candidate     │      │  Explainability Agent    │      │ Database           │   │   SME Dashboard   │
│ profiles      │      └─────────────────────────┘      └──────────────────┘   └─────────────────┘
└──────────────┘
```

This mirrors the architecture slide in the submission deck. The hackathon
prototype simplifies four conceptual agent *roles* into two Claude API calls
(JD parsing, and a combined evaluate+rank+explain call) to keep latency and
cost reasonable — see "Productionizing" below for how this splits apart at
scale.

### Why this scoring approach
Each candidate gets three sub-scores, combined into a composite:

```
composite_score = 0.5 × skill_fit_score
                + 0.3 × experience_score
                + 0.2 × practical_fit_score
```

`skill_fit_score` and `experience_score` come from the model reasoning over
the candidate's actual words, not keyword overlap. `practical_fit_score`
captures hard constraints from the JD — location, language, transport —
which matter enormously in non-metro hiring and are usually ignored by
generic ATS keyword filters.

Every score ships with an `evidence_quote`: a short phrase copied verbatim
from the candidate's own profile, so a hiring SME can sanity-check the
reasoning in one glance instead of trusting a black box.

---

## Running the demo right now

The `index.html` file calls `fetch('https://api.anthropic.com/v1/messages')`
directly from the browser with no API key in the code — this works because
Claude.ai's artifact runtime authenticates that call for you. **This is the
fastest way to demo the working prototype**: open the file as a Claude.ai
artifact, edit the sample JD/candidates if you like, and hit "Run."

This is also genuinely good for screen-recording your submission video —
the agent pipeline trace at the top lights up in real time as each agent
runs.

## Running it standalone (GitHub Pages, your own domain, judging outside Claude.ai)

Browsers can't call `api.anthropic.com` directly from an arbitrary domain
(no CORS/auth) — you need a small backend in between. That's what
`backend/` is for:

```bash
cd backend
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...
uvicorn main:app --reload --port 8000
```

Test it:
```bash
curl -X POST http://localhost:8000/rank \
  -H "Content-Type: application/json" \
  -d @sample_data.json
```

Then point the frontend at your backend instead of `api.anthropic.com`
directly — swap the two `fetch()` calls in `index.html`'s `callClaude()`
function for a single call to `POST /rank` on your deployed backend, and
remove the client-side prompt strings (they move server-side, where your
API key already lives).

---

## Productionizing (what we'd do with more than a 42-day sprint)

- **Real retrieval, not a stub.** `CandidateRetriever` in `agents.py` is a
  lexical-overlap placeholder so the prototype runs with zero external
  services. Swap in multilingual sentence embeddings (IndicBERT/LaBSE-class)
  and a vector DB (FAISS/Pinecone/Weaviate) for real semantic retrieval
  across regional languages and large candidate pools.
- **Voice input.** Add an ASR step (Whisper or Bhashini) ahead of the JD
  Parser Agent so JDs and profiles can come in as raw voice notes, which is
  how a lot of non-metro hiring actually happens today.
- **Confidence & validation.** Flag low-confidence transcriptions and
  contradictory profiles for manual review instead of silently ranking them
  — see the deck's Explainability & Data Validation slide.
- **SME delivery.** Wire the ranked shortlist into a WhatsApp Business API
  bot so a shop owner with no HR team gets the result where they already
  are, instead of a dashboard they'd need training to use.

---

## Track 1 submission checklist

- [x] System architecture (see deck + diagram above)
- [x] JD understanding & candidate evaluation logic
- [x] Ranking methodology with explainability
- [x] Working prototype (`index.html`)
- [x] Deployable backend skeleton (`backend/`)
- [ ] Demo video — record yourself running `index.html` as a Claude.ai
      artifact, walking through a JD, then swapping to the second sample JD
      to show the ranking *change* (Anjali ranks differently once "no bike"
      stops being a constraint — that's a great 10-second demo beat)
- [ ] Add this repo's URL and your video link to the submission deck
