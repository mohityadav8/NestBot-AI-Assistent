## 🤖 NestBot AI Assistant - Implementation Plan

> This issue tracks my implementation plan for the NestBot AI Assistant as part of the 
> OWASP Nest Sponsorship Program 2026. Reference: #908

---

## 📌 Approach

Rather than building NestBot as a standalone service, I plan to extend the existing 
Django backend by adding a dedicated `nestbot` app - integrating with the `slack-bolt` 
and `OpenAI` libraries already present in the codebase.

**Core stack (already in Nest backend):**
- `slack-bolt` - Slack event handling
- `OpenAI` - LLM response generation  
- `pytest` - test coverage
- Django ORM - `BotInteraction` model for logging and feedback

---

## 🗂️ Project Structure

```
backend/
└── nestbot/
    ├── __init__.py
    ├── apps.py
    ├── handlers/
    │   ├── events.py       # message, app_mention, reaction_added
    │   └── commands.py     # /owasp help, /owasp ask
    ├── classifier/
    │   ├── model.py        # intent classifier (DistilBERT / TF-IDF)
    │   └── channel_map.py  # OWASP project → Slack channel routing
    ├── llm/
    │   └── responder.py    # OpenAI integration + system prompt
    ├── models.py           # BotInteraction model
    └── tests/
        ├── test_handlers.py
        ├── test_classifier.py
        └── test_responder.py

docs/
└── nestbot/
    ├── scenarios.md        # 30+ question types + expected responses
    └── maintainer.md       # setup, config, rollback plan
```

---

## 📅 Timeline

### Week 1–2 — Slack integration
- [ ] Create `nestbot` Django app
- [ ] Set up `slack-bolt` with Socket Mode (dev) + Events API (prod)
- [ ] Handle `message`, `app_mention`, `reaction_added` events
- [ ] Implement `/owasp help` and `/owasp ask <query>` slash commands
- [ ] Slack signing secret verification middleware
- [ ] `BotInteraction` model for interaction logging
- [ ] `pytest` unit tests for all event handlers

### Week 3–4 — Intent classification
- [ ] Collect + annotate training data from OWASP Slack history (links in #908)
- [ ] Build two-stage classifier:
  - Stage 1: Is it OWASP-related? (binary, runs before OpenAI call)
  - Stage 2: Category — `general_owasp` / `project_specific` / `membership` / `gsoc` / `off_topic`
- [ ] Train DistilBERT or TF-IDF + logistic regression classifier
- [ ] Channel routing map (Juice Shop, ZAP, WebGoat, Nest, SAMM, DSOMM...)
- [ ] Document evaluation metrics (accuracy, precision, recall)

### Week 5–6 — LLM response generation
- [ ] OWASP-specific system prompt using Nest Django model data + owasp.org FAQs
- [ ] Thread-aware replies via `thread_ts` (never top-level)
- [ ] Graceful fallback for low-confidence queries
- [ ] OpenAI token usage logging per interaction
- [ ] 👍/👎 reaction feedback stored in `BotInteraction`

### Week 7–8 — Testing + staged deployment
- [ ] 30+ scenario matrix → `docs/nestbot/scenarios.md`
- [ ] Full `pytest` suite (unit + integration, ≥80% coverage)
- [ ] UAT in `#project-nest-bot-testing`
- [ ] Structured logging: accuracy, fallback rate, token usage, reaction scores
- [ ] Staged rollout:
  - `#project-nest-bot-testing` → OWASP project channels → `#owasp-community`
- [ ] Maintainer docs + rollback plan → `docs/nestbot/`

---

## ✅ Deliverables

| # | Deliverable | Detail |
|---|---|---|
| 1 | `nestbot` Django app | Merged into Nest backend, full test coverage |
| 2 | Intent classifier | Trained model with evaluation metrics |
| 3 | Scenario matrix | 30+ question types, responses, channel routing |
| 4 | `pytest` test suite | Unit + integration, ≥80% coverage on new code |
| 5 | Logging dashboard | Accuracy, fallback rate, token usage, reactions |
| 6 | Staged deployment | Rollout plan + documented rollback |
| 7 | Maintainer docs | Full docs in `docs/nestbot/` |

---

## 🔗 Reference

- Tracking issue: #908
- My reference project (similar architecture): https://github.com/AnthropicBots/hiero-bot-py
- Working demo: https://github.com/AnthropicBots/hiero-bot-py/pull/7
- GitHub: https://github.com/mohityadav8
