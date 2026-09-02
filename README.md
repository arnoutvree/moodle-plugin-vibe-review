# Moodle Plugin Vibe Review

An AI coding agent skill that reviews a git diff or a set of files in a **Moodle plugin** against Moodle's own coding, security and API conventions — before you commit or open a pull request.

It's built for the "vibe coding" workflow: you don't have to write Moodle plugin code yourself to ship it responsibly. Clear requirements and a feature that works are not enough on their own — an explicit review gate, checked against Moodle's real API and security rules, is what turns AI-assisted development into something you can actually trust.

It's a single markdown instruction file, so it works with whatever AI coding agent you use — natively as a [Claude Code](https://claude.com/product/claude-code) Skill, or pasted into any other agent's system prompt / rules file / custom instructions.

## What it checks

- Input handling with the correct `PARAM_*` types (`optional_param()` / `required_param()`)
- Database access through `$DB`, never raw concatenated SQL
- Output escaping (`format_string()`, `format_text()`, `s()`)
- CSRF protection (`require_sesskey()`, `sesskey()` in forms/URLs)
- Access control (`require_login()` + the right `require_capability()` per entry point)
- Privacy API (GDPR) coverage for any new personal-data storage
- Events API usage for meaningful state changes
- Deprecated Moodle API calls for the plugin's declared minimum version
- General web security (OWASP Top 10), applied through a Moodle lens
- AI/LLM-specific risks (prompt injection, excessive agency, hidden context exposure) when the plugin itself calls an LLM
- A "sad path" check — do new tests cover failure cases, not just the happy path?

It never edits code — it produces a severity-ranked report (critical / warning / info), and a fix is always an explicit next step.

It's a fast, conversational **complement** to Moodle's official [`moodle-plugin-ci`](https://moodledev.io/general/development/tools/pluginci) (PHPCS with `moodle-cs`, PHPUnit, Behat) — not a replacement. Use `moodle-plugin-ci` as your authoritative, automatable gate; use this skill for a quick pass while you're still iterating, and for things a linter won't catch (a missing `sesskey()` check, missing Privacy API coverage, a questionable API choice).

## Install

Clone this repo:

```bash
git clone https://github.com/arnoutvree/moodle-plugin-vibe-review.git
```

**Claude Code:** symlink the skill folder into its user-level skills directory:

```bash
ln -s "$(pwd)/moodle-plugin-vibe-review/skill/moodle-plugin-vibe-review" ~/.claude/skills/moodle-plugin-vibe-review
```

**Any other AI coding agent** (Cursor, Windsurf, Copilot, a custom agent, etc.): point it at [`skill/moodle-plugin-vibe-review/SKILL.md`](skill/moodle-plugin-vibe-review/SKILL.md) directly, or copy its contents into whatever instruction-file convention that agent uses (`AGENTS.md`, `.cursorrules`, a system prompt).

## Usage

From inside a Moodle plugin's working directory:

```
review my changes
```

In Claude Code you can also call it explicitly:

```
/moodle-plugin-vibe-review
```

Point it at specific files instead of the current diff by naming them in your prompt.

If the plugin has its own `README.md` / `CLAUDE.md` / `CONTRIBUTING.md` with coding standards, the skill reads those first and treats them as authoritative alongside the general Moodle conventions above.

## License

MIT — see [LICENSE](LICENSE).
