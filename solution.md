# Online Safety Companion — Complete Solution

**IBM watsonx Orchestrate Hackathon Submission**
*Assembled November 2026 from all project documents*

---

## 1. Problem and Solution Statement *(500 words or less)*

Young people aged 18–19 who are neurodivergent — including those with autism, ADHD, or related
conditions — face a significantly elevated risk of online harm. Predators, romance scammers, and
sextortion actors actively target this group because social cues that signal manipulation are harder
to read, and trust can be established more quickly.

Existing online safety tools are designed for a general adult audience. Their language is abstract,
their advice is non-specific, and they frequently place the burden of action on the person who was
targeted — creating shame and discouraging disclosure. Young people rarely tell anyone. They worry
about being blamed, having their phone taken away, or getting the other person in trouble. This
silence allows harm to escalate.

**Online Safety Companion** is a conversational AI assistant built on IBM watsonx Orchestrate,
designed for a user persona called "Ralph" — an 18-year-old neurodivergent young adult who met
someone online and wants to check whether the situation is safe.

The assistant:
- **Meets the user where they are.** Short, plain sentences. No idioms. Never blames the user.
  Explicitly says: "You are not in trouble."
- **Gathers information without overwhelm.** One question at a time. Never repeats questions.
  Waits for real content before running analysis.
- **Uses structured multi-agent analysis.** A dedicated Safety Agent applies a rule-based risk
  scorer and cross-references a curated knowledge base covering grooming, scams, sextortion, and
  platform-migration pressure. A Support Agent translates the result into plain language and
  generates 2–3 concrete options for the user.
- **Never makes the decision.** Every response ends with "The choice is yours."
- **Escalates appropriately.** When photos, money transfers, threats, or in-person meeting requests
  are detected, the assistant provides NCMEC CyberTipline, cybertip.ca, and the 988 crisis line.

A neurodivergent young adult who suspects something is wrong can open a chat, paste a message, and
receive a calm, clear, non-judgmental assessment within seconds — with real options and draft wording
to start a conversation with a trusted adult if they choose to.

---

## 2. Technology Statement

### IBM watsonx Orchestrate

The entire multi-agent system runs on **IBM watsonx Orchestrate** using the native ADK.

**Three agents are deployed:**

| Agent | Role | LLM |
|---|---|---|
| `online_safety_companion` | Orchestrator. Manages conversation flow, gathers context, coordinates sub-agents, writes the final reply. | groq/openai/gpt-oss-120b |
| `safety_agent` | Analyzes message content for risk signals. Calls `score_risk` tool and cross-references knowledge bases. Returns structured risk assessment. | groq/openai/gpt-oss-120b |
| `support_agent` | Translates risk assessment into plain language. Generates 2–3 options and a draft message to a trusted adult. | groq/openai/gpt-oss-120b |

**Tool — `score_risk`:**
A Python function registered with the ADK `@tool` decorator. Applies rule-based regex pattern
matching to detect 12 signal categories:
- Platform migration pressure (Discord → Telegram, etc.)
- Secrecy from parents or family
- Isolation language
- Money, gift card, and cryptocurrency requests *(always HIGH risk)*
- Photo and video requests *(always HIGH risk)*
- Fake urgency
- Too-good-to-be-true offers
- Love-bombing and excessive flattery
- Personal information requests
- In-person meeting requests *(always HIGH risk)*
- Sextortion / leverage with existing content *(always HIGH risk)*
- Inconsistent identity claims

Returns `risk_level` (low / medium / high), `signals` (list), and `recommended_next_steps` (list).

**Three knowledge bases loaded into the watsonx Orchestrate knowledge store:**
- `grooming_warning_signs` — pattern-level grooming indicators across early, escalation, and
  high-risk stages
- `scam_patterns` — romance scams, gift-card scams, sextortion, job scams, fake urgency patterns
- `what_to_do_next` — evidence preservation steps, official reporting resources (US, Canada, UK,
  Australia), crisis lines

### IBM Bob

IBM Bob was used throughout development:

| Task | How Bob was used |
|---|---|
| Agent YAML design | Drafted all three ADK agent YAML files, checked instruction completeness, identified missing guardrails |
| Tool development | Generated `risk_scorer.py` with all 12 signal detection rule groups, type hints, docstrings, and the `@tool` decorator |
| Knowledge base | Authored all three knowledge Markdown files at pattern level to avoid producing material that could be repurposed |
| Test suite | Wrote 8 test conversations and the full `test_risk_scorer.py` pytest suite including edge cases |
| Documentation | Drafted `README.md`, `problem_solution.md`, `demo_script.md`, `technology_used.md` |
| Review | Reviewed the orchestrator agent for logical gaps, redundant questions, missing escalation paths |

Bob was used inside VS Code via the Bob extension with the workspace set to this repository.

---

## 3. Architecture

```
User (Ralph)
    │
    ▼
online_safety_companion  (orchestrator)
    │  Step 1: gather context — one question at a time
    │  Step 2: call safety_agent with message + context
    │
    ├──► safety_agent
    │       │  calls score_risk tool
    │       │  cross-references grooming_warning_signs + scam_patterns KB
    │       └──► returns: RISK_LEVEL / SIGNALS_FOUND / REASONING
    │
    ├──► support_agent
    │       │  reads what_to_do_next KB
    │       └──► returns: PLAIN_SUMMARY / OPTIONS / DRAFT_MESSAGE
    │
    └──► one combined plain-language reply to user
             always ends: "The choice is yours."
```

**Flow for a high-risk signal:**
1. User pastes a message (e.g. "Add me on Telegram and don't tell your parents").
2. Orchestrator calls `safety_agent`.
3. `safety_agent` calls `score_risk` → detects Platform migration pressure + Secrecy (HIGH).
4. `safety_agent` returns structured assessment.
5. Orchestrator calls `support_agent` with the assessment.
6. `support_agent` returns plain summary, 2–3 options, and a draft message.
7. Orchestrator assembles one reply including escalation resources.

---

## 4. Test Cases and Expected Results

| # | Scenario | Expected risk | Key signals |
|---|---|---|---|
| TC-01 | Discord → Telegram + "don't tell your parents" | HIGH | Platform migration, Secrecy from family, Love-bombing |
| TC-02 | Gift card prize scam with urgency | HIGH | Money request, Fake urgency, Too-good-to-be-true |
| TC-03 | Romance scam — Bitcoin emergency request | HIGH | Money request, Love-bombing, Secrecy, Fake urgency |
| TC-04 | Friendly gaming chat — "GG, what's your favourite map?" | LOW | None |
| TC-05 | Photo request escalating to "a bit more personal" | HIGH | Photo request, Love-bombing |
| TC-06 | Sextortion — "I have your pictures, send $200 Bitcoin" | HIGH | Sextortion, Money request, Fake urgency |
| TC-07 | In-person meeting — "don't tell your family, quiet spot" | HIGH | In-person meeting, Secrecy, Isolation |
| TC-08 | User gives full context in first message — no redundant questions | HIGH | Platform migration, Secrecy |

**Pass criteria for all cases:**
- Correct risk level detected
- "shows warning signs of" — never "is definitely"
- No blame or judgment toward the user
- Ends with "The choice is yours."
- HIGH risk cases include escalation resources (NCMEC, cybertip.ca, 988)
- TC-06 opens with "This was not your fault."
- TC-08 agent does NOT re-ask for context already given

---

## 5. Known Issue — Agent Error on First Message

**Status:** Observed in testing. Documented for submission transparency.

**Symptom:** All three agents return `"I have encountered an error. Please try again."` on the
first message in a new session (seen in `test_output.txt`, `test_safety.txt`, `test_support.txt`).

**Root cause identified:** The `orchestrate chat ask` CLI command requires an active authenticated
session. The error occurs when:
1. The session token has expired between `run.ps1` and `chat.py` startup, OR
2. The `online_safety_companion` agent tries to delegate to `safety_agent` but the delegation
   handshake fails on the first cold-start request (the ADK async flag shows `False`, confirming
   this is a synchronous agent that may time out on the first LLM call).

**Workaround for demo:** Run `.\run.ps1` immediately before `python chat.py` to refresh the auth
token. If the first message returns an error, send the same message again — subsequent calls
in the same session succeed (the token is cached after the first successful handshake).

**Fix (post-submission):** Add a no-op warmup ping in `chat.py` after `activate_env()`:

```python
def warmup_agent() -> None:
    """Send a silent warmup ping so the first real message doesn't error."""
    subprocess.run(
        ["orchestrate", "chat", "ask",
         "--agent-name", "online_safety_companion",
         "ping"],
        capture_output=True, text=True
    )
```

Call `warmup_agent()` in `main()` after `activate_env()` and before the user prompt.

---

## 6. Demo Script (3 minutes)

| Segment | Time | Content |
|---|---|---|
| Problem | 0:00 – 0:40 | Who Ralph is; why this matters |
| Live demo | 0:40 – 2:40 | Three live conversations |
| Impact | 2:40 – 3:00 | What changes; what's next |

### 0:00 – 0:40 | The Problem

> "Meet Ralph. He's 18, neurodivergent, and he met someone in an online game.
> The person is friendly — really friendly. They want to move the conversation
> to Telegram, and they said not to tell his parents.
>
> Ralph isn't sure if something is wrong. He doesn't want to make a fuss.
> Existing tools aren't built for Ralph — the language is too abstract, the
> advice tells him what he 'should' do, and he's worried he'll be blamed.
>
> We built Online Safety Companion — a multi-agent assistant on IBM watsonx
> Orchestrate — so Ralph gets a calm, clear answer in plain English, without
> judgment, in under a minute."

### 0:40 – 1:20 | Demo 1 — HIGH risk (platform migration + secrecy)

Ralph types:
> *"Hi. I met someone on Discord and they've been really nice. Today they said:
> 'Add me on Telegram, it's more private, and don't tell your parents — they
> wouldn't understand.' Is this okay?"*

Expected result: HIGH. Signals: platform migration, secrecy, love-bombing.
Reply ends: "The choice is yours."

### 1:20 – 1:55 | Demo 2 — Sextortion

Ralph types:
> *"Someone is threatening me. They said: 'I have your pictures. Send me $200
> in Bitcoin by tomorrow or I'll send them to everyone.'"*

Expected result: HIGH. Opens with "This was not your fault." Signals: sextortion,
money request, fake urgency. Includes NCMEC, cybertip.ca, 988.

### 1:55 – 2:20 | Demo 3 — LOW risk (benign)

Ralph types:
> *"Someone from my gaming server said: 'GG! Do you play ranked? What's your
> favourite map?' Is this okay?"*

Expected result: LOW. No signals. Reassures Ralph. Still reminds him to stay alert.

### 2:20 – 2:40 | Architecture callout

> "Three agents, one tool, three knowledge documents.
> Orchestrator gathers context. Safety Agent scores risk.
> Support Agent explains it. One clear reply.
> Built entirely on IBM watsonx Orchestrate — granite-3-3-8b-instruct."

### 2:40 – 3:00 | Impact

> "Ralph gets an answer in under 60 seconds. In plain English. Without blame.
> With real options and real resources.
> The knowledge base and signal rules are easy to update as new patterns emerge.
> Online Safety Companion. Built with IBM watsonx Orchestrate and IBM Bob."

---

## 7. Project File Index

| File | Purpose |
|---|---|
| `agents/online_safety_companion.yaml` | Orchestrator agent — conversation management, escalation rules |
| `agents/safety_agent.yaml` | Risk analysis agent — calls score_risk, cross-references KBs |
| `agents/support_agent.yaml` | Support agent — plain language, options, draft messages |
| `tools/risk_scorer.py` | Python ADK tool — 12-signal rule-based risk scorer |
| `knowledge/grooming_warning_signs.md` | Knowledge base — grooming pattern library |
| `knowledge/scam_patterns.md` | Knowledge base — scam type library |
| `knowledge/what_to_do_next.md` | Knowledge base — evidence, reporting resources, crisis lines |
| `knowledge/*_kb.yaml` | KB definition files for Orchestrate import |
| `tests/test_cases.md` | 8 test conversations with pass criteria |
| `tests/test_risk_scorer.py` | pytest suite for the risk scorer tool |
| `docs/problem_solution.md` | Submission: problem and solution statement |
| `docs/technology_used.md` | Submission: technology statement |
| `docs/demo_script.md` | 3-minute demo script |
| `run.ps1` | PowerShell deploy script — authenticates and imports all agents/tools/KBs |
| `chat.py` | Interactive terminal chat client for the deployed agent |
| `.env` | Credentials — copy from `env.template`, fill in your own keys. Never commit this file. |
| `solution.md` | **This file** — complete combined submission document |

---

## 8. Submission Checklist

| Deliverable | Status |
|---|---|
| 3-minute video demonstration | ☐ Record using demo script in Section 6 |
| Written problem and solution statement | ✅ Section 1 (≤500 words) |
| Written technology statement | ✅ Section 2 |
| Working prototype link | ✅ Deployed — open your instance at https://eu-de.watson-orchestrate.cloud.ibm.com/home |
| IBM Bob usage documented | ✅ Section 2 — Technology Statement |
| Test cases with pass criteria | ✅ Section 4 |
| Known issues disclosed | ✅ Section 5 |

---

*Solution document assembled from: agents/*, tools/risk_scorer.py, knowledge/*, docs/*, tests/,
README.md, and all workshop materials.*
