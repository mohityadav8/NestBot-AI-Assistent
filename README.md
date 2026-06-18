# NestBot AI Assistant — Implementation Plan
> ref: [#908](https://github.com/OWASP/Nest/issues/908) · [@mohityadav8](https://github.com/mohityadav8)

---

## Codebase Audit — What Already Exists

| Component | File | Status |
|---|---|---|
| Slack Bolt app handler | `backend/apps/slack/` | ✅ exists |
| Slash commands (`/ai`, `/owasp`, `/gsoc`, `/contribute` …) | `backend/apps/slack/commands/` | ✅ exists |
| `CommandBase` + `EventBase` inheritance pattern | `backend/apps/slack/commands/command.py` | ✅ exists |
| Event handlers (`team_join`, `member_joined_channel`, `app_mention`) | `backend/apps/slack/events/` | ✅ exists |
| `Message`, `Member`, `Conversation`, `Workspace` models | `backend/apps/slack/models/` | ✅ exists |
| `slack_sync_messages` management command | `backend/apps/slack/management/commands/` | ✅ exists |
| Channel ID constants (`OWASP_PROJECT_*`, `OWASP_GSOC_CHANNEL_ID` …) | `backend/apps/slack/constants.py` | ✅ exists |
| `OWASP_KEYWORDS` set | `backend/apps/slack/constants.py` | ✅ exists |
| `MessagePosted` event handler | `backend/apps/slack/events/message_posted.py` | ✅ exists |
| `QuestionDetector` — RAG + OpenAI OWASP relevance filter | `backend/apps/slack/common/question_detector.py` | ✅ exists |
| `AgenticRAGAgent` — LangGraph retrieve → generate → evaluate loop | `backend/apps/ai/agent/agent.py` | ✅ exists |
| `Retriever` — pgvector cosine similarity on `Chunk` model | `backend/apps/ai/agent/tools/rag/retriever.py` | ✅ exists |
| `Generator` — GPT-4o answer generation | `backend/apps/ai/agent/tools/rag/generator.py` | ✅ exists |
| `generate_ai_reply_if_unanswered` — RQ background job | `backend/apps/slack/services/message_auto_reply.py` | ✅ exists |
| `Conversation.is_nest_bot_assistant_enabled` flag | `backend/apps/slack/models/conversation.py` | ✅ exists |
| Latest migration | `backend/apps/slack/migrations/0022_…py` | ✅ exists |

### What's genuinely missing

- No `reaction_added` handler — 👍/👎 feedback loop from issue spec not implemented
- No `BotInteraction` model — nowhere to log AI replies, confidence, or reaction outcomes
- No channel routing — bot answers everything OWASP-related but never redirects
- No `/owasp ask <query>` subcommand — only `/ai` exists
- No scenario matrix or maintainer docs
- Staged deployment not configured (`#project-nest-bot-testing` gate missing)

---

## Week 1–2 — `BotInteraction` model + reaction feedback handler

**Goal:** Close the feedback loop gap. Every AI reply gets logged; every 👍/👎 reaction gets recorded.

- [ ] Create `BotInteraction` model with fields: `channel_id`, `user_id`, `user_message`, `bot_response`, `intent_category`, `confidence_score`, `thumbs_up` (null = no reaction yet), `tokens_used`, `slack_reply_ts`, `created_at`
  - `backend/apps/slack/models/bot_interaction.py` ← **new file**

- [ ] Register `BotInteraction` in models `__init__.py`
  - `backend/apps/slack/models/__init__.py` ← extend

- [ ] Generate migration `0023_bot_interaction.py`
  - `backend/apps/slack/migrations/0023_bot_interaction.py` ← **new file**

- [ ] Hook into `message_auto_reply.py` — after `client.chat_postMessage`, create a `BotInteraction` row with `slack_reply_ts` set to the reply's `ts`
  - `backend/apps/slack/services/message_auto_reply.py` ← extend

- [ ] Create `reaction_added/` event package — `bot_feedback.py` checks if the reacted-to message `ts` matches a `BotInteraction.slack_reply_ts`, then sets `thumbs_up = True/False` on 👍/👎; ignores all other reactions
  - `backend/apps/slack/events/reaction_added/__init__.py` ← **new file**
  - `backend/apps/slack/events/reaction_added/bot_feedback.py` ← **new file**

- [ ] Register `reaction_added` handler in `events/__init__.py` `configure_slack_events()`
  - `backend/apps/slack/events/__init__.py` ← extend

- [ ] Register `BotInteraction` in Django admin
  - `backend/apps/slack/admin/__init__.py` ← extend

- [ ] Write pytest tests: `BotInteraction` model, reaction handler, `message_auto_reply` hook
  - `backend/tests/unit/apps/slack/events/reaction_added/bot_feedback_test.py` ← **new file**
  - `backend/tests/unit/apps/slack/services/message_auto_reply_test.py` ← extend

---

## Week 3–4 — Channel router — redirect instead of answering

**Goal:** Bot knows when to redirect rather than answer. Saves tokens, better UX.

- [ ] Build `channel_router.py` — `PROJECT_CHANNEL_MAP` using existing `constants.py` IDs (`OWASP_PROJECT_JUICE_SHOP_CHANNEL_ID`, `OWASP_GSOC_CHANNEL_ID`, etc.); keyword-based routing returns `(channel_id, label)` or `None` for general questions
  - `backend/apps/slack/services/channel_router.py` ← **new file**

- [ ] Hook router into `message_auto_reply.py` — call router before running the full RAG pipeline; if a match is found, post a redirect message instead of generating an AI answer
  - `backend/apps/slack/services/message_auto_reply.py` ← extend

- [ ] Store `intent_category` and `routed_to_channel` in the `BotInteraction` row
  - `backend/apps/slack/services/message_auto_reply.py` ← extend

- [ ] Add `/owasp ask <query>` subcommand — wire it to `get_blocks()` in `handlers/ai.py` using the existing `Owasp.find_command()` dispatch pattern
  - `backend/apps/slack/commands/owasp.py` ← extend
  - `backend/apps/slack/templates/commands/owasp.jinja` ← extend

- [ ] Write pytest tests for `channel_router` covering all `PROJECT_CHANNEL_MAP` entries + fallback `None` case
  - `backend/tests/unit/apps/slack/services/channel_router_test.py` ← **new file**

- [ ] Write scenario-based integration tests for `/owasp ask` command
  - `backend/tests/unit/apps/slack/commands/owasp_test.py` ← extend

---

## Week 5–6 — System prompt hardening + scenario matrix

**Goal:** Make AI responses actually good for OWASP community Slack.

- [ ] Write OWASP Slack-specific system prompt — grounded in `Project`/`Chapter` models, concise tone, links not walls of text. Store as new `Prompt` key `nestbot-slack-system-prompt` so Arkadii can edit it from Django admin without a deploy
  - `backend/apps/core/models/prompt.py` ← extend (add `get_nestbot_slack_system_prompt()`)

- [ ] Wire new prompt key into `generator.py` — use `nestbot-slack-system-prompt` when invoked from Slack context; fall back to `rag-system-prompt`
  - `backend/apps/ai/agent/tools/rag/generator.py` ← extend

- [ ] Build scenario matrix doc — 30+ rows: user message | router decision | expected bot response

  Example rows:

  | User message | Router decision | Expected response |
  |---|---|---|
  | "how do I contribute to OWASP?" | `general_owasp` | contribution guide + #contribute link |
  | "what is juice shop?" | `project_specific → #project-juiceshop` | redirect, don't answer |
  | "how do I apply for GSoC?" | `gsoc → #gsoc` | redirect + gsoc guide link |
  | "what's the weather today?" | `off_topic` | silently ignored by `QuestionDetector` |
  | "who maintains the cheat sheet series?" | `general_owasp` | RAG answer from project data |
  | "is there a chapter in Berlin?" | `general_owasp` | RAG answer from chapter data |
  | "how do I report a security issue?" | `general_owasp` | security policy link |

  - `docs/nestbot/scenarios.md` ← **new file**

- [ ] Add pytest parametrize test class using scenario matrix rows as the test harness
  - `backend/tests/unit/apps/slack/services/scenario_test.py` ← **new file**

- [ ] Verify ≥80% coverage on all new code added in weeks 1–6
  - `backend/tests/` ← coverage pass

---

## Week 7–8 — Staged deployment + maintainer docs

**Goal:** Ship without breaking `#owasp-community`.

- [ ] UAT in `#project-nest-bot-testing` — set `is_nest_bot_assistant_enabled = True` on that `Conversation` via Django admin; test full flow with Arkadii
  - `backend/apps/slack/models/conversation.py` ← extend (admin action)

- [ ] Add structured logging to `message_auto_reply.py`: response latency, token cost per interaction, redirect rate, feedback score trend
  - `backend/apps/slack/services/message_auto_reply.py` ← extend

- [ ] Write maintainer doc covering: how to enable the bot on a new channel, how to edit the system prompt without a deploy, rollback steps (`is_nest_bot_assistant_enabled = False` silences the bot immediately), monitoring queries against `BotInteraction`
  - `docs/nestbot/maintainer.md` ← **new file**

- [ ] Final coverage pass — all new files ≥80%
  - `backend/tests/` ← coverage pass

- [ ] Staged rollout in order: `#project-nest-bot-testing` → OWASP project channels → `#owasp-community` (flip `Conversation` flags via Django admin in sequence)
  - `backend/apps/slack/models/conversation.py` ← admin

---

## Deliverables summary

| # | Deliverable | Location | Type |
|---|---|---|---|
| 1 | `BotInteraction` model + migration 0023 | `backend/apps/slack/models/` | new |
| 2 | `reaction_added` feedback handler | `backend/apps/slack/events/reaction_added/` | new |
| 3 | Channel router service | `backend/apps/slack/services/channel_router.py` | new |
| 4 | Hooks into `message_auto_reply.py` | `backend/apps/slack/services/` | extend |
| 5 | `/owasp ask <query>` subcommand | `backend/apps/slack/commands/owasp.py` | extend |
| 6 | OWASP Slack system prompt (admin-editable) | `Prompt` model, new key | extend |
| 7 | pytest suite ≥80% coverage on new code | `backend/tests/unit/apps/slack/` | new |
| 8 | 30+ scenario matrix | `docs/nestbot/scenarios.md` | new |
| 9 | Staged deployment + rollback plan | `docs/nestbot/maintainer.md` | new |

---

## Architecture reference

```
Slack message / @mention / /owasp ask <query>
         │
         ▼
QuestionDetector (gpt-4o-mini)
  is this OWASP-related?  ──NO──▶  ignore silently
         │ YES
         ▼
channel_router.py (keyword match)
  known project/channel?  ──YES──▶  post redirect + log BotInteraction(routed)
         │ NO
         ▼
AgenticRAGAgent (LangGraph)
  retrieve  ──▶  generate  ──▶  evaluate
                                    │ complete?  ──YES──▶  post reply
                                    │ NO  ──▶  refine (max 3 iterations)
         │
         ▼
BotInteraction logged (slack_reply_ts stored)
         │
         ▼
reaction_added event (👍 / 👎)
  thumbs_up updated on BotInteraction row
```

---

*Implements [#908](https://github.com/OWASP/Nest/issues/908)*
