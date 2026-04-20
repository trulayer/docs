# Docs Tasks

Parent: [TRU-87](https://linear.app/omnimoda/issue/TRU-87)

## Definition of Done

Docs are content + nav + a working preview — `mint dev` renders the page without errors, and `mint broken-links` passes.

This repo doesn't ship runtime code, so the backend/frontend "tests required" rule doesn't apply here. It is replaced by two rules:

- **`mint broken-links` passes** before every merge.
- Any PR that removes or renames a page must also remove/update every link to it.

## Status

### Scaffolding
- [x] Mintlify starter cloned
- [x] `docs.json` customised for TruLayer — nav, colors, logo, socials
- [x] Starter example content removed
- [x] `api-reference/openapi.yaml` seeded from `backend/api/openapi.yaml`

### Content — shipped this round
- [x] `index.mdx` — landing
- [x] `quickstart.mdx` — install + first trace in Python + TypeScript
- [x] `concepts/overview.mdx`
- [x] `concepts/traces-and-spans.mdx`
- [x] `concepts/sessions.mdx`
- [x] `concepts/evals.mdx`
- [x] `concepts/feedback.mdx`
- [x] `concepts/metrics.mdx`
- [x] `sdks/python/overview.mdx`
- [x] `sdks/python/tutorial.mdx` — full 13-step walkthrough
- [x] `sdks/python/configuration.mdx`
- [x] `sdks/python/reference.mdx`
- [x] `sdks/typescript/overview.mdx` — stub + link to TRU-92
- [x] `api-reference/introduction.mdx` — auth, base URL, errors, rate limits
- [x] `development.mdx` — contributor guide

### Content — pending (tracked in Linear)

- [ ] TypeScript SDK — tutorial, configuration, reference → [TRU-92](https://linear.app/omnimoda/issue/TRU-92)
- [ ] Dashboard guide — one page per `(dashboard)` route → [TRU-94](https://linear.app/omnimoda/issue/TRU-94)
- [ ] Framework integrations — OpenAI, Anthropic, LangChain, Vercel AI, PydanticAI → [TRU-95](https://linear.app/omnimoda/issue/TRU-95)
- [ ] Best practices — tracing patterns, PII, cost/latency, sampling, error handling → [TRU-96](https://linear.app/omnimoda/issue/TRU-96)
- [ ] Glossary → [TRU-96](https://linear.app/omnimoda/issue/TRU-96)

### Infrastructure — pending

- [ ] `package.json` with `sync-openapi` script → part of [TRU-88](https://linear.app/omnimoda/issue/TRU-88)
- [ ] CI on backend repo to open a PR here when `api/openapi.yaml` changes → [TRU-93](https://linear.app/omnimoda/issue/TRU-93)
- [ ] Mintlify GitHub app installed + custom domain `docs.trulayer.ai` → [TRU-98](https://linear.app/omnimoda/issue/TRU-98)
- [ ] Logo + favicon assets replaced with real brand files (currently starter placeholders)

### Cross-repo rule rollout — pending

- [ ] Docs-update rule added to `frontend/CLAUDE.md`, `backend/CLAUDE.md`, and both SDK repos → [TRU-97](https://linear.app/omnimoda/issue/TRU-97)
- [ ] Docs-update rule added to `frontend/docs/conventions.md` and `backend/docs/conventions.md`
- [ ] PR template updated in all four repos to include a docs checklist item

## Local dev

```bash
npm i -g mint
mint dev            # preview at localhost:3000
mint broken-links   # run before every merge
```
