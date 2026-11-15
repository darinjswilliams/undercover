# Pilot plan

## Scope and approach

Build the pilot as a small local FastAPI and SQLite application, seeded from the operational files in `data/`. The backend is the only path by which the agent reads claims, policies, customers, policy wording and its own instructions. It will expose explicit, typed operations for the dashboard and agent, each with an `operation_id`, a one-line docstring and no optional inputs.

The database will not read, copy or expose `data/evals/`, and the agent will use `data/policy_wording.md` rather than the claims manual. Payments remain simulated records only. A payout above the configured $1,000 limit will be impossible to record until an authorized handler signs it off, including when the request originates from the agent.

Implementation will be split into five phases so that the data layer, safety boundary, agent, interface and whole application can each be checked before the next layer depends on it.

## Phase 1: Seeded claims records and read API

Create the SQLite schema and repository layer for the queue, claim details, policies, customers, handlers, policy wording, decisions, activity and simulated payments. On an empty local database, load only the pilot operational records from `data/`; keep the generated database out of source control. Define Pydantic response models and add read-only FastAPI operations for the queue, an individual claim, linked policy and customer records, policy wording, handlers, agent instructions and a claim's recorded activity.

Queue responses will provide the claim identifier, customer, summary, amount and one of the business states `waiting`, `in progress`, `waiting for sign-off` or `decided`. Claim-detail responses will contain the complete claim, its linked records, recorded decision, activity and payment state so the dashboard does not have to infer business state. Unknown identifiers return a clear 404 response.

Success criteria:

- [ ] Backend unit tests create an isolated database, seed it from the permitted operational files, and prove that a queue item, its policy, customer and policy wording can be retrieved with their expected identifiers and fields.
- [ ] Backend tests prove that an unknown claim, policy or customer returns HTTP 404, and that the seed source never includes an `evals` path or evaluation record.
- [ ] `uv run pytest backend -q` exits 0 and its coverage report includes the new database and read-route code.
- [ ] With the backend running, `curl http://127.0.0.1:8000/api/claims` returns HTTP 200 and a JSON queue containing `CLM-1001` with status `waiting`; `curl http://127.0.0.1:8000/api/claims/CLM-1001` returns the claim, its policy and customer context, and an empty decision.
- [ ] `curl http://127.0.0.1:8000/api/policy-wording` returns the policy wording version and text, while a request for `/api/claims/does-not-exist` returns HTTP 404.

## Phase 2: Claim workflow and payment guardrail

Add write operations and validation for recording an agent's progress, decision, payout calculation, policy section and two-to-three-sentence explanation; starting a claim; signing off a claim; and recording a simulated payment with an explicit amount. Store an immutable activity sequence for meaningful workflow steps so the dashboard can show what happened while the agent worked.

Starting a claim moves it to `in progress`. Recording an approval over the company approval limit moves it to `waiting for sign-off`; all other completed decisions move it to `decided`. A sign-off requires an existing over-limit approval and a handler whose record permits it. The separate payment operation accepts an amount, confirms that the claim was approved and that it does not exceed the approved amount, and rejects an over-limit payment before sign-off. It records no real payment or external request.

Success criteria:

- [ ] Backend tests cover starting a claim, recording approve, decline and refer decisions, and the resulting queue state and ordered activity entries.
- [ ] Backend tests show that a $1,000-or-less approval can be paid only for its approved amount or less, while a payout above $1,000 is rejected before sign-off without creating a payment record.
- [ ] Backend tests show that an unauthorized handler cannot sign off, an authorized handler can sign off exactly once, and only then can the approved over-limit amount be recorded as paid.
- [ ] Every newly added FastAPI operation has a one-line docstring, an explicit `operation_id`, typed request and response models, and no optional request inputs; route tests exercise its success and error cases.
- [ ] `uv run pytest backend -q` exits 0, and direct API calls demonstrate a `waiting for sign-off` response before sign-off, a rejected payment response before sign-off, and a successful payment response after authorized sign-off.

## Phase 3: UnderCover Agent and backend-only tools

Replace the hello-world prompt and health tool with a concise claims-handling instruction set. It will tell the agent to use the policy wording, decide approve/decline/refer, calculate any approved payout, cite the relevant policy section and give a two- or three-sentence explanation. It will explicitly exclude the claims manual and decision answers.

Implement LangChain tools for every required read and workflow action. Each tool will call the corresponding backend operation through HTTP rather than loading files or importing backend data. The agent's entry point remains `run(claim_id, thread_id)`: it starts the claim, fetches the required context through tools, records progress steps, submits the decision, and attempts the distinct payment operation only through the safety-checked API. The existing model selection remains `MODEL_NAME` when supplied and `openai/gpt-6-luna` at low reasoning effort otherwise.

Success criteria:

- [ ] Agent tests use scripted stub-model messages only; no test requires an API key or calls a model.
- [ ] Tests verify that every agent tool makes an HTTP request to the backend API, that the agent can retrieve a claim and all required context, record progress, and submit a structured decision with its explanation and policy section.
- [ ] A scripted over-limit approval test proves the agent's payment attempt is rejected before handler sign-off and that the claim remains `waiting for sign-off`; a separate signed-off test proves the payment tool can then succeed.
- [ ] Tests retain the default model and `MODEL_NAME` override behavior, including low reasoning effort only for the default model.
- [ ] `uv run pytest agent -q` exits 0, and `uv run pytest -q` exits 0 with no live model invocation.

## Phase 4: Claims-handler dashboard

Replace the health page with a responsive React dashboard styled with the UnderCover brand: Instrument Sans for UI copy, JetBrains Mono for claim identifiers and figures, sea green/marigold/coral state accents, light rounded panels and plain US business language. It will have a queue view, claim-detail view and read-only instructions view.

The queue will make all four business states clear and let a handler open a claim. The detail screen will show the full claim story, documents, policy, customer claim history, agent decision and policy citation, live activity timeline, simulated payment status and only the actions permitted by that claim's state. In particular, it will show authorized handlers an explicit sign-off control for over-limit approvals and will not offer a payment action until the safety conditions are met. The instructions screen will fetch and display the same read-only instructions used by the agent.

Success criteria:

- [ ] Vitest tests mock API responses and cover loading, error and populated queue states; navigating to a claim; decision and activity rendering; instructions display; and the permitted sign-off/payment controls.
- [ ] `npm run check:ts` exits 0 and reports TypeScript, lint, formatting and frontend coverage success at or above the configured 80% threshold.
- [ ] Using the Playwright MCP browser against `npm run dev`, the queue shows the claims and clearly labelled waiting, in-progress, waiting-for-sign-off and decided states; opening a claim shows its story, documents, policy/customer data and activity timeline.
- [ ] In the browser, an over-limit approved claim shows that payment is awaiting sign-off; after an authorized sign-off, the payment control becomes available and its successful simulated-payment result is shown.
- [ ] In the browser, the instructions view is readable and has no edit control; visual inspection confirms the documented colors, fonts, rounded panels, no heavy gradients and US-dollar formatting.

## Phase 5: End-to-end coverage, documentation and release check

Replace the health-only Playwright scenario with pilot user journeys that do not require a live model: browse the seeded queue, inspect a claim and agent activity, open the read-only instructions, and exercise the high-value approval/sign-off/payment boundary through the UI. Keep the health check as a lightweight smoke test if it remains useful. Update the short project README only where needed to run the pilot, and update `docs/STATUS.md` with what has been built and the current commands.

Run the complete cross-platform check command from a clean application state, fix any failures without weakening checks or coverage, tick the completed criteria in this plan as each phase is verified, and commit each completed phase as the implementation process requires.

Success criteria:

- [ ] Playwright tests cover queue-to-detail navigation, visible activity and decision evidence, the read-only instructions screen, and the blocked-until-sign-off payment rule; `npm run e2e` exits 0.
- [ ] `docs/STATUS.md` accurately describes the backend, agent, dashboard, tests and how to start the pilot; it does not claim that real payments, sign-in or the claims manual are implemented.
- [ ] `npm run check` exits 0: Ruff format/lint, Pyright, pytest with at least 80% coverage, Prettier, ESLint, TypeScript, Vitest with at least 80% coverage, and Playwright all pass.
- [ ] Final manual browser check with the Playwright MCP server confirms a handler can understand the queue, inspect a complete decision and its reasons, watch the recorded agent steps, view instructions without editing them, and cannot pay an over-limit claim before sign-off.
