# Codex instructions

This is the public TruLayer developer documentation repo. `CLAUDE.md` is the detailed source of truth; read it before making any non-trivial change.

## Scope

- Mintlify docs for `docs.trulayer.ai`.
- Public-only content: quickstarts, SDK guides, API reference, dashboard guides, integrations, concepts, best practices, and changelog.
- The OpenAPI copy in `api-reference/openapi.yaml` is generated from the backend OpenAPI source of truth.

## Working rules

- Make changes on a feature/fix branch and open a PR to `main`. Never commit directly to `main`.
- Do not publish private paths, planning URLs, deployment details, private runbooks, or sibling-repo implementation notes.
- Use `https://api.trulayer.ai` in public examples unless a page is explicitly about local development.
- Never hand-edit generated API reference content. Sync it from the backend spec.
- Update `docs.json` when adding, moving, or removing pages.

## Style

- Use active voice and second person.
- Keep sentences concise.
- Use sentence case for headings.
- Bold UI labels, for example: Click **Settings**.
- Use code formatting for commands, paths, file names, env vars, fields, and enum values.

## Verification

Run before opening a PR:

```bash
pnpm broken-links
```

For new or heavily edited pages, also preview locally:

```bash
pnpm dev
```

To sync API reference:

```bash
pnpm sync-openapi
```
