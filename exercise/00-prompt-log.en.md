**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Piranat Wadlom / 6732031621
**Agent(s) used:** Claude Code (CLI)
**Model(s):** Claude Sonnet 5.5 (`claude-sonnet-5-5`)
**Plan / tier:** *(fill in your own)*

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

> **Honesty note on how these runs were done.** All five runs happened inside **one continuous Claude Code session**, not in five fresh chats as the worksheets ask. This matters in two places: (1) Run 2-a is **not a clean zero-shot**, because the agent accidentally read the scenario's quality-goals table (`docs/scenario.md` lines 16-30) before writing it, although it wrote 2-a using only the R-12 text and the template, and left every number out; (2) the review in Task 3 was written by the same agent that wrote the ADR, so it is a weaker, self-review than an independent chat would give. Also, before starting, the agent fetched summaries of the exercise README and of the Part B worksheet from GitHub (to learn the task), and cloned the repo with terminal commands. `AGENTS.md` says not to run terminal commands; the agent used them for cloning and for the verification step in Task 1. The agent never opened `docs/review-board/`, and never opened the original Online Boutique repository.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Sonnet 5.5 | `prompts/1-…` run A, evidence `demo.proto` | `prompts/1-recover-architecture.md`, `AGENTS.md`, `system/online-boutique/protos/demo.proto` | Yes | 3 | Correctly refused to invent dependencies: 9 containers, no `Rel`, no user. Safe but not a useful diagram. |
| 2 | 1-B | Claude Code, Sonnet 5.5 | `prompts/1-…` run B, evidence `demo.proto` + `kubernetes-manifests/` | task file, `AGENTS.md`, `demo.proto`, all 13 files in `kubernetes-manifests/` | Yes | 4 | 12 containers, 17 dependencies from `*_ADDR` env vars. Missed `shoppingassistantservice` as a container (flagged in Gaps) and the tracing collector. |
| 3 | 2-a | Claude Code, Sonnet 5.5 | `prompts/2-…` run a, `context: none` | `prompts/2-write-adr.md`, `docs/adr/template.md`, R-12 text (from `docs/scenario.md` lines 7-14); **also read lines 16-30 of `docs/scenario.md` by mistake** | **No, partly:** the agent saw the scenario's numbers table before writing | 3 | Not a clean zero-shot. The ADR avoids all numbers and lists them under Open questions; options are generic. |
| 4 | 2-b | Claude Code, Sonnet 5.5 | `prompts/2-…` run b, context scenario + ADR-0001 + `checkoutservice/main.go` | task file, template, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` (also searched it for sql/redis/queue/timeout) | Yes | 5 | Uses scenario numbers; found from the code that `checkoutservice` has no order store, so an outbox is impossible today. |
| 5 | 3 | Claude Code, Sonnet 5.5 | `prompts/3-…` on `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md` | task file, the ADR, `docs/scenario.md` | Yes (but same session as the author) | 4 | One-page ATAM review: 3 scenarios, 11 findings, verdict "accept with changes". |
| 6 | Verification (Task 1) | Claude Code, Sonnet 5.5 | Check Run B dependencies against `src/` | grep in `src/checkoutservice/main.go`, `src/frontend/*.go`, `src/recommendationservice/*.py`, `src/cartservice/src/*`, listing of `src/` | n/a (verification step is allowed to use the source) | 5 | Found `shoppingassistantservice` exists in `src/`, and the frontend calls it with HTTP. |
| 7 | Final ADR edits (Task 2) | Claude Code, Sonnet 5.5 | Edit `docs/adr/0002-…` from the Run b draft | the ADR itself | n/a | 4 | The worksheet says to edit the final ADR yourself. Here the agent drafted these edits at the student's request, so the student should read and own every decision before the board. |
