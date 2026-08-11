APM_RULES {

## Documentation

- Prefer linking to existing authoritative documentation over writing new explanatory content when a canonical source already covers the topic (e.g. skogai-routing's own reference docs for `@`-link conventions).

## Testing

- Use Node's built-in `node:test` / `node:assert` for tests in this repo (`npm test` runs `node --test build/`). No external test framework dependency.

## Version Control

- Base branch: `master`.
- Branch naming: `type/short-description` (e.g. `feat/link-placeholders`).
- Commit convention: Conventional Commits, `type(scope): description` or `type: description` when no scope applies. Observed types in this repo's history: `feat`, `fix`, `docs`, `chore`. Add `refactor`/`test` when applicable.

} //APM_RULES
