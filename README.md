# [NestBot] AI-Powered Auto-Responder - Implementation Plan (ref: #908)

> **Submitted by:** [@mohityadav8](https://github.com/mohityadav8) | OWASP Nest Sponsorship Program 2026
> **Reference issue:** https://github.com/OWASP/Nest/issues/908
> **Related project:** [Hiero Bot](https://github.com/AnthropicBots/hiero-bot-py) - similar LLM+webhook architecture I already built

---

## What already exists in `apps.slack`

Before writing a single line of code, I studied the existing codebase. Here's what's already there:

| Component | Location | Status |
|---|---|---|
| `slack-bolt` App handler | `backend/apps/slack/` | ✅ exists |
| Slash commands (`/projects`, `/gsoc`, `/ai`, `/contribute`...) | `backend/apps/slack/commands/` | ✅ exists |
| `CommandBase` + `EventBase` inheritance pattern | `backend/apps/slack/commands/command.py` | ✅ exists |
| Event handlers (`team_join`, `member_joined_channel`) | `backend/apps/slack/events/` | ✅ exists |
| Jinja template system for Slack Block Kit | `backend/apps/slack/templates/` | ✅ exists |
| `Message`, `Member`, `Conversation`, `Workspace` models | `backend/apps/slack/models/` | ✅ exists |
| `slack_sync_messages` management command | `backend/apps/slack/management/commands/` | ✅ exists |
| Channel ID constants | `backend/apps/slack/constants.py` | ✅ exists |
| OpenAI integration | `backend/` (pyproject.toml) | ✅ exists |

**What does NOT exist yet (the actual work for #908):**
- `message` event handler for AI auto-response (the bot currently does NOT respond to free-form messages)
- Intent classifier to route/filter queries before hitting OpenAI
- `BotInteraction` model for logging responses + feedback
- 👍/👎 `reaction_added` feedback loop
- OWASP-grounded system prompt for the `/ai` command expansion
- Scenario matrix and test coverage for the above

---

## Scope of my contribution

I will add the following **inside the existing `apps.slack`** - no new Django app, no new infrastructure - following the exact patterns Arkadii established:

```
backend/apps/slack/
├── events/
│   ├── message/                          ← NEW
│   │   ├── __init__.py
│   │   ├── owasp_community.py            ← NEW: AI auto-responder for #owasp-community
│   │   └── catch_all.py                  ← NEW: fallback for other channels
│   └── reaction_added/                   ← NEW
│       └── bot_feedback.py               ← NEW: 👍/👎 feedback handler
├── classifier/                           ← NEW
│   ├── __init__.py
│   ├── intent.py                         ← NEW: two-stage intent classifier
│   └── channel_map.py                    ← NEW: extends existing constants.py routing
├── models/
│   └── bot_interaction.py                ← NEW: logs every AI response + feedback score
├── migrations/
│   └── 0022_bot_interaction.py           ← NEW: auto-generated
├── templates/
│   └── events/
│       └── message/
│           ├── ai_response.jinja         ← NEW: Block Kit template for AI answers
│           └── channel_redirect.jinja    ← NEW: Block Kit template for routing
└── tests/unit/apps/slack/
    ├── test_message_handler.py           ← NEW
    ├── test_classifier.py                ← NEW
    └── test_bot_interaction.py           ← NEW
```

---

## Week-by-week plan

### Week 1–2 - `message` event handler + `BotInteraction` model

**Goal:** Bot receives free-form messages and logs them. No AI yet — just the plumbing.

- Add `message` event listener following the existing `EventBase` pattern in `backend/apps/slack/events/event.py`
- Register it in `backend/apps/slack/events/__init__.py` (same pattern as `team_join`, `member_joined_channel`)
- Add `reaction_added` event handler for 👍/👎 feedback - stores reaction against `BotInteraction` record
- Create `BotInteraction` Django model:
  ```python
  # backend/apps/slack/models/bot_interaction.py
  class BotInteraction(models.Model):
      channel_id = models.CharField(max_length=64)
      user_id = models.CharField(max_length=64)
      user_message = models.TextField()
      bot_response = models.TextField()
      intent_category = models.CharField(max_length=64)
      confidence_score = models.FloatField()
      thumbs_up = models.BooleanField(null=True)   # null = no reaction yet
      tokens_used = models.IntegerField(default=0)
      created_at = models.DateTimeField(auto_now_add=True)
  ```
- Generate and apply migration `0022_bot_interaction.py`
- Write `pytest` tests for the new event handlers using mocked Slack payloads (following patterns in `backend/tests/unit/apps/slack/`)

**Deliverable:** Bot receives messages, logs them to `BotInteraction`, stores feedback reactions.

---

### Week 3–4 - Intent classifier

**Goal:** Classify every message before calling OpenAI. Cheap, fast, accurate routing.

**Two-stage design:**
- **Stage 1 - OWASP relevance filter** (binary): Is this even OWASP-related? If no → ignore silently. This runs before any OpenAI call to keep costs zero on off-topic messages.
- **Stage 2 - Category routing**: Classifies into `general_owasp` / `project_specific` / `membership` / `chapter` / `gsoc` / `off_topic`

**Implementation:**
- Start with TF-IDF + logistic regression (fast, interpretable, no GPU needed)
- Training data: annotate messages from the 3 OWASP Slack links in issue #908 + existing `Message` model data already synced via `slack_sync_messages`
- Build `channel_map.py` extending the existing `OWASP_*_CHANNEL_ID` constants already in `backend/apps/slack/constants.py`:
  ```python
  # extends existing constants — doesn't replace them
  PROJECT_CHANNEL_MAP = {
      "juice_shop":  "C...",   # #project-juiceshop
      "zap":         "C...",   # #project-zap
      "webgoat":     "C...",   # #project-webgoat
      "nest":        "C07JLLG2GFQ",  # already in constants.py
      "samm":        "C...",
      "dsomm":       "C...",
  }
  ```
- Document classifier evaluation metrics (accuracy, precision, recall) on a held-out test set
- Expose classifier as internal service callable by message event handler

**Deliverable:** Trained classifier with documented metrics, channel routing map covering all major OWASP projects.

---

### Week 5–6 - AI response generation

**Goal:** Wire OpenAI into the message flow with an OWASP-specific system prompt.

**Implementation:**
- Use existing OpenAI setup already in the Nest backend (same client the `/ai` slash command uses)
- Write OWASP-specific system prompt grounded in:
  - Live data from existing Django models (Project, Chapter, Contribution opportunities)
  - owasp.org documentation
  - Annotated Slack message history from Week 3–4
- Implement thread-aware replies using `thread_ts` - bot always replies in-thread, never as top-level message (matches community UX expectations)
- Add graceful fallback: if classifier confidence is low → respond with `"I'm not sure — try asking in #owasp-community"` rather than hallucinating
- Log `tokens_used` per interaction in `BotInteraction`
- Create Jinja templates for AI responses following existing template structure in `backend/apps/slack/templates/`

**Deliverable:** End-to-end working AI response flow. Bot receives message → classifies → calls OpenAI → posts threaded reply → logs interaction.

---

### Week 7–8 - Testing, scenario matrix, staged deployment

**Goal:** Harden, document, deploy.

- Write `docs/nestbot/scenarios.md` - 30+ question types with expected outputs:
  ```
  | User message | Intent | Expected response |
  |---|---|---|
  | "how do I contribute to owasp?" | general_owasp | contribute guide + #contribute link |
  | "what is juice shop?" | project_specific | project summary + #project-juiceshop |
  | "how do i join gsoc?" | gsoc | gsoc guide + #gsoc |
  | "what's the weather today?" | off_topic | silently ignored |
  ```
- Full `pytest` suite:
  - Unit tests for message handler, classifier, OpenAI integration
  - Integration tests with mocked Slack payloads (following existing test patterns)
  - Target: ≥80% coverage on all new code
- UAT in `#project-nest-bot-testing` - collect feedback from the team
- Set up structured logging: response accuracy, fallback rate, token cost, reaction scores
- Staged rollout:
  - `#project-nest-bot-testing` → OWASP project channels → `#owasp-community`
- Write `docs/nestbot/maintainer.md` - setup, config, rollback plan

**Deliverable:** Full test suite, scenario matrix, staged deployment, maintainer docs.

---

## Deliverables summary

| # | Deliverable | Location |
|---|---|---|
| 1 | `message` + `reaction_added` event handlers | `backend/apps/slack/events/` |
| 2 | `BotInteraction` model + migration | `backend/apps/slack/models/` |
| 3 | Intent classifier with evaluation metrics | `backend/apps/slack/classifier/` |
| 4 | Channel routing map (extends existing constants) | `backend/apps/slack/classifier/channel_map.py` |
| 5 | OWASP system prompt + Jinja response templates | `backend/apps/slack/templates/events/message/` |
| 6 | `pytest` suite ≥80% coverage on new code | `backend/tests/unit/apps/slack/` |
| 7 | 30+ scenario matrix | `docs/nestbot/scenarios.md` |
| 8 | Staged deployment + rollback plan | `docs/nestbot/maintainer.md` |

---

## Why I'm the right person for this

I recently built **[Hiero Bot](https://github.com/AnthropicBots/hiero-bot-py)** - a GitHub automation tool with:
- LLM-powered PR review (same OpenAI integration pattern)
- Event-driven webhook handling (GitHub webhooks → same as Slack events)
- Automated routing model (6-signal PR health scoring → same as intent classifier)

The architecture is identical. The platform changes from GitHub to Slack; the pattern doesn't.

I've also studied the `apps.slack` codebase in detail before writing this plan — every file path and pattern above references the actual existing code, not generic guesses.

**GitHub:** https://github.com/mohityadav8
**Hiero Bot:** https://github.com/AnthropicBots/hiero-bot-py
**Working demo PR:** https://github.com/AnthropicBots/hiero-bot-py/pull/7
**Working demo issue:** https://github.com/AnthropicBots/hiero-bot-py/issues/6 
---

*Closes / implements #908*
