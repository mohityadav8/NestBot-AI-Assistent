# [NestBot] AI-Powered Auto-Responder - Implementation Plan (ref: #908)

> **Submitted by:** [@mohityadav8](https://github.com/mohityadav8) | OWASP Nest Sponsorship Program 2026
> **Reference issue:** https://github.com/OWASP/Nest/issues/908
> **Related project:** [Hiero Bot](https://github.com/AnthropicBots/hiero-bot-py) - similar LLM+webhook architecture I already built

---

## What's already in `apps.slack` (full audit)

I went through the codebase before writing this. There's more built than the issue description implies - some of the core AI plumbing is already wired up:

| Component | Location | Status |
|---|---|---|
| `slack-bolt` App handler | `backend/apps/slack/` | ✅ exists |
| Slash commands (`/projects`, `/gsoc`, `/ai`, `/contribute`...) | `backend/apps/slack/commands/` | ✅ exists |
| `CommandBase` + `EventBase` inheritance pattern | `backend/apps/slack/commands/command.py` | ✅ exists |
| Event handlers (`team_join`, `member_joined_channel`, `app_mention`) | `backend/apps/slack/events/` | ✅ exists |
| `Message`, `Member`, `Conversation`, `Workspace` models | `backend/apps/slack/models/` | ✅ exists |
| `slack_sync_messages` management command | `backend/apps/slack/management/commands/` | ✅ exists |
| Channel ID constants (community, gsoc, sponsorship, etc.) | `backend/apps/slack/constants.py` | ✅ exists |
| **`MessagePosted` event handler** — handles free-form messages | `backend/apps/slack/events/message_posted.py` | ✅ exists |
| **`QuestionDetector`** — RAG + OpenAI binary OWASP relevance filter | `backend/apps/slack/common/question_detector.py` | ✅ exists |
| **`AgenticRAGAgent`** — full RAG answer generation | `backend/apps/ai/agent/agent.py` | ✅ exists |
| **`generate_ai_reply_if_unanswered`** — RQ background job | `backend/apps/slack/services/message_auto_reply.py` | ✅ exists |
| `Conversation.is_nest_bot_assistant_enabled` flag | `backend/apps/slack/models/conversation.py` | ✅ exists |
| Tests for `message_posted`, `question_detector`, `message_auto_reply` | `backend/tests/unit/apps/slack/` | ✅ exists |

The bot already receives messages, runs the OWASP relevance check, queues a background job, and posts a threaded AI reply if nobody answers within `QUEUE_RESPONSE_TIME_MINUTES`. That's the main loop from #908.

**What's genuinely missing:**

- No `reaction_added` handler - the 👍/👎 feedback loop from the issue spec doesn't exist anywhere
- No `BotInteraction` model - there's nowhere to log AI responses, confidence, or reaction outcomes
- No channel routing - the bot responds to everything OWASP-related but never redirects ("this sounds like a #project-juiceshop question")
- No scenario matrix or maintainer docs
- Staged deployment hasn't been set up (`#project-nest-bot-testing` gate doesn't exist)

---

## What I'm actually building

Since the message handling and intent detection are already there, I'm not going to duplicate them. The scope is:

```
backend/apps/slack/
├── events/
│   └── reaction_added/                   ← NEW
│       ├── __init__.py
│       └── bot_feedback.py               ← NEW: 👍/👎 handler, updates BotInteraction
├── models/
│   └── bot_interaction.py                ← NEW: logs AI responses + feedback
├── migrations/
│   └── 0023_bot_interaction.py           ← NEW: auto-generated
└── services/
    └── channel_router.py                 ← NEW: extends existing constants.py routing

docs/nestbot/
├── scenarios.md                          ← NEW: 30+ scenario matrix
└── maintainer.md                         ← NEW: setup, config, rollback
```

Plus extending two existing files:
- `message_posted.py` - add `BotInteraction` logging after the RQ job is enqueued
- `message_auto_reply.py` - call `channel_router` before generating an AI reply, so we can redirect instead of answering when the bot is confident about the target channel

---

## Week-by-week plan

### Week 1–2 - `BotInteraction` model + `reaction_added` feedback handler

**Goal:** Close the feedback loop gap. Every AI reply gets logged, every reaction gets recorded.

- Create `BotInteraction` model:

```python
# backend/apps/slack/models/bot_interaction.py
class BotInteraction(models.Model):
    channel_id         = models.CharField(max_length=64)
    user_id            = models.CharField(max_length=64)
    user_message       = models.TextField()
    bot_response       = models.TextField()
    intent_category    = models.CharField(max_length=64, blank=True)
    confidence_score   = models.FloatField(null=True)
    thumbs_up          = models.BooleanField(null=True)  # null = no reaction yet
    tokens_used        = models.IntegerField(default=0)
    slack_reply_ts     = models.CharField(max_length=32, blank=True)
    created_at         = models.DateTimeField(auto_now_add=True)
```

- Generate migration `0023_bot_interaction.py`
- Hook into existing `message_auto_reply.py` - after `client.chat_postMessage`, create a `BotInteraction` row with `slack_reply_ts` set to the reply's `ts`
- Add `reaction_added` event handler following the same `EventBase` pattern as `app_mention.py`:
  - Checks if the reacted-to message `ts` matches a `BotInteraction.slack_reply_ts`
  - If so, sets `thumbs_up = True/False` depending on the emoji
  - Ignores reactions on non-bot messages
- Register in `events/__init__.py` alongside the existing handlers
- Write pytest tests using mocked Slack payloads (following the patterns in `message_posted_test.py`)

**Deliverable:** Every AI reply is logged. Reactions on bot replies update the feedback record.

---

### Week 3-4 - Channel routing

**Goal:** Bot knows when to redirect rather than answer.

Right now `QuestionDetector` does a binary OWASP relevance check (yes/no). That's good, but the issue also asks for channel routing — if someone asks about Juice Shop in #owasp-community, the bot should point them to `#project-juiceshop` rather than generating a full AI answer.

**Implementation:**

- Build `services/channel_router.py` — takes a message and returns the most relevant channel redirect (if any), or `None` if the question is genuinely general:

```python
# extends existing OWASP_PROJECT_*_CHANNEL_ID constants — doesn't replace them
PROJECT_CHANNEL_MAP = {
    "juice_shop":    OWASP_PROJECT_JUICE_SHOP_CHANNEL_ID,
    "nest":          OWASP_PROJECT_NEST_CHANNEL_ID,
    "nettacker":     OWASP_PROJECT_NETTACKER_CHANNEL_ID,
    "threat_dragon": OWASP_PROJECT_THREAT_DRAGON_CHANNEL_ID,
    "gsoc":          OWASP_GSOC_CHANNEL_ID,
    "chapter":       OWASP_ASKOWASP_CHANNEL_ID,
    "jobs":          OWASP_JOBS_CHANNEL_ID,
    "contribute":    OWASP_CONTRIBUTE_CHANNEL_ID,
}
```

- Start simple - keyword matching against existing `OWASP_KEYWORDS` set in `constants.py` plus project names. Lightweight, no extra OpenAI call.
- Hook into `message_auto_reply.py`: if router returns a channel, post a redirect message ("Sounds like this belongs in #project-juiceshop!") instead of running the full RAG pipeline. Saves tokens, better UX.
- Store `intent_category` and matched channel in the `BotInteraction` row
- Tests: scenario-based unit tests covering the 30+ cases from the scenario matrix

**Deliverable:** Bot redirects project-specific questions to the right channel instead of trying to answer everything.

---

### Week 5-6 - System prompt hardening + scenario matrix

**Goal:** Make the AI responses actually good for OWASP-specific questions.

The current `AgenticRAGAgent` uses a generic prompt (pulled via `Prompt.get_slack_question_detector_prompt()`). For community-facing replies, it needs OWASP grounding.

- Write an OWASP-specific system prompt for the Slack reply context — different from the question detection prompt. Grounds responses in:
  - Existing Django models: `Project`, `Chapter`, available through the RAG retriever
  - Standard OWASP contribution paths
  - Tone expectations for a community Slack (concise, links not walls of text)
- Wire it in as a separate `Prompt` key so Arkadii can edit it from the admin without a deploy
- Build `docs/nestbot/scenarios.md` — 30+ question types with expected bot behaviour:

```
| User message | Router decision | Expected response |
|---|---|---|
| "how do I contribute to owasp?" | general_owasp | contribution guide + #contribute link |
| "what is juice shop?" | project_specific → #project-juiceshop | redirect, don't answer |
| "how do i apply for gsoc?" | gsoc → #gsoc | redirect + gsoc guide link |
| "what's the weather today?" | off_topic | silently ignored (QuestionDetector filters) |
| "who maintains the cheat sheet series?" | general_owasp | RAG answer from project data |
```

- Use scenarios as the test harness — each row becomes a pytest parametrize case

**Deliverable:** Better responses, documented behaviour, test suite anchored to real scenarios.

---

### Week 7-8 - Staged deployment + maintainer docs

**Goal:** Ship it without breaking #owasp-community.

- UAT in `#project-nest-bot-testing` — set `is_nest_bot_assistant_enabled = True` only on that conversation object, test the full flow with the team
- Add structured logging: response latency, token cost per interaction, redirect rate, feedback score trend
- Write `docs/nestbot/maintainer.md`:
  - How to enable the bot for a new channel (the `Conversation` admin flag)
  - How to edit the system prompt without a deploy
  - Rollback: disable via `is_nest_bot_assistant_enabled = False`, bot goes silent immediately
  - Monitoring queries against `BotInteraction` (thumbs_up rate, token spend)
- Final coverage pass — target ≥80% on all new code
- Staged rollout: testing channel → OWASP project channels → `#owasp-community`

**Deliverable:** Production-ready, with a clear way to roll back if something goes wrong.

---

## Deliverables summary

| # | What | Where |
|---|---|---|
| 1 | `BotInteraction` model + migration 0023 | `backend/apps/slack/models/` |
| 2 | `reaction_added` feedback handler | `backend/apps/slack/events/reaction_added/` |
| 3 | Channel router service | `backend/apps/slack/services/channel_router.py` |
| 4 | Hooks into existing `message_auto_reply.py` | extends existing file |
| 5 | OWASP system prompt (admin-editable) | `Prompt` model, new key |
| 6 | pytest suite ≥80% coverage on new code | `backend/tests/unit/apps/slack/` |
| 7 | 30+ scenario matrix | `docs/nestbot/scenarios.md` |
| 8 | Staged deployment + rollback plan | `docs/nestbot/maintainer.md` |

---

## Why I'm a good fit

I built **[Hiero Bot](https://github.com/AnthropicBots/hiero-bot-py)** — a GitHub App that does PR health scoring, AI code review via Anthropic API, and contributor onboarding automation. The architecture is basically the same thing: webhook event → intent classification → LLM call → structured reply → feedback tracking. Different platform, same pattern.

More relevant here: I read the `apps.slack` code before writing this, not after. The existing `MessagePosted` → `QuestionDetector` → `generate_ai_reply_if_unanswered` chain is solid — I'm not proposing to replace it, just extend it with the pieces that are actually missing.

**GitHub:** https://github.com/mohityadav8
**Hiero Bot:** https://github.com/AnthropicBots/hiero-bot-py
**Working demo PR:** https://github.com/AnthropicBots/hiero-bot-py/pull/7
**Working demo issue:** https://github.com/AnthropicBots/hiero-bot-py/issues/6

---

*Implements #908*
