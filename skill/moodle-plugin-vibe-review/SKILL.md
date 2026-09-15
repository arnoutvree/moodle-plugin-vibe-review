---
name: moodle-plugin-vibe-review
description: |
  Reviews a local git diff or a set of files in a Moodle plugin against Moodle's own coding, security and API conventions (coding style, capability checks, sesskey/CSRF, PARAM_* input cleaning, output escaping, Privacy API) before you commit or open a PR. Reads the plugin's own CLAUDE.md/README/docs for project-specific rules first, then adds Moodle-platform knowledge, and delivers a severity-ranked review report — never edits code itself.

  Use for requests like "review my changes", "check this diff against Moodle conventions", "moodle-plugin-vibe-review", "review my code before I commit", or before opening a pull request on a Moodle plugin.

created: 2026-09-02
---

# Moodle Plugin Vibe Review

A manual, on-demand alternative to a CI/CD security gate: instead of an automated pipeline, you call this skill yourself in chat — for example right before a commit or a PR — to check that AI-assisted ("vibe coded") changes to a Moodle plugin actually meet Moodle's own conventions.

This skill exists because a plugin owner who doesn't write the code themselves still needs a way to trust it. Clear requirements and a working feature are not enough on their own — an explicit review gate, checked against Moodle's real API and security rules, is what makes AI-assisted plugin development something you can rely on.

## Scope: what this skill is and isn't

- **Is:** a Moodle-specific check that adds platform knowledge (Moodle coding style, API conventions, common security pitfalls) to a code review.
- **Isn't:** a replacement for a general-purpose code review skill that checks broad correctness, readability or design. Use a generic review for that; use this skill specifically to check whether Moodle plugin code follows Moodle's own rules. The two can be run back to back.
- **Isn't:** a substitute for Moodle's official [Automated plugin testing / `moodle-plugin-ci`](https://moodledev.io/general/development/tools/pluginci) (PHPCS with `moodle-cs`, PHPUnit, Behat, etc.) — this skill is a fast, conversational first pass; `moodle-plugin-ci` remains the authoritative, automatable check.

---

## Phase 1: determine plugin context

1. Read the plugin's own `README.md` / `CLAUDE.md` / `AGENTS.md` / `docs/` if present, for the plugin type (e.g. `mod`, `local`, `block`, `report`, `auth`), supported Moodle version range, and any project-specific rules the plugin maintainer has already written down.
2. If the plugin was built with the [`moodle-plugin-development`](https://github.com/arnoutvree/moodle-plugin-development) process, also read `specs.md`, `user-stories.md` and `plan.md` when present — not to re-judge scope or design (that's the Spec/Test-axis comparison the plugin-development process itself does, separately from this skill), but because they often already commit to Moodle-specific choices this skill checks anyway: the `specs.md` Capabilities table (expected capability names/contextlevel/archetypes) and its Phase 2e privacy-by-design commitments (legal basis, opt-out, retention — check these actually landed in `classes/privacy/provider.php`), and `plan.md`'s architecture decisions (e.g. Hooks API vs `lib.php` callbacks, capability syntax, table/field naming). Treat these as authoritative project-specific rules alongside `README.md`/`CLAUDE.md`.
3. Check `version.php` for `$plugin->component` and `$plugin->requires` — this tells you the plugin's frankenstyle name and minimum Moodle version, which matters for which APIs are safe to use (e.g. `PARAM_ALPHANUMEXT` vs older `PARAM_ALPHANUM`, deprecated session/API calls).
4. No project-specific rules found? That's a normal outcome — fall back to Moodle's general conventions below. Don't guess at plugin-specific business rules; ask the user if something is ambiguous.

---

## Phase 2: gather the standards to check against

**Always, as a baseline — Moodle-specific secure coding:**

- **Input handling:** `optional_param()` / `required_param()` with the narrowest correct `PARAM_*` type (e.g. `PARAM_ALPHANUMEXT` for values that may contain digits — `PARAM_ALPHA` silently strips them, which is a common source of subtle bugs). Never read `$_GET`/`$_POST`/`$_REQUEST` directly.
- **Database access:** always through `$DB` (`get_records`, `get_record_sql` with placeholders, `execute()` with parameters) — never string-concatenated SQL. Use `$DB->sql_like()` for portable LIKE queries.
- **Output escaping:** `format_string()` / `format_text()` for anything that may contain user input or HTML, `s()` or `format_string()` for plain text in attributes, Mustache templates over raw string concatenation where the plugin already uses them.
- **CSRF protection:** `require_sesskey()` on every state-changing action reachable via GET/POST, `sesskey()` included in forms and action URLs.
- **Access control:** `require_login()` plus the correct `require_capability()` check for every entry point — check against the plugin's own `db/access.php` capability definitions, not just a login check.
- **Privacy API (GDPR):** any plugin that stores personal data needs a `classes/privacy/provider.php` implementing the relevant privacy interfaces (`core_userlist_provider`, etc.) — flag new personal-data storage that isn't reflected there. If `specs.md` has a Phase 2e privacy-by-design section, check its legal basis/opt-out/retention commitments actually landed in the code, not just that a provider file exists.
- **Events / logging:** state changes that other code or admins would reasonably want to observe should trigger a Moodle event (`\component\event\...`) rather than only writing to a custom table.
- **Deprecated APIs:** flag calls to Moodle APIs marked deprecated for the plugin's declared `$plugin->requires` version (check `lib/deprecatedlib.php`-style deprecation notices or the [Moodle DevDocs deprecation list](https://moodledev.io/general/releases) for that version) — these typically still work but log a `debugging()` notice and will break in a future major version.
- **General web security (OWASP Top 10):** applied through a Moodle lens — e.g. treat unescaped output as XSS, unsanitised SQL fragments as injection, missing capability checks as broken access control. Label findings that rest on general OWASP knowledge rather than a Moodle-specific rule as such, so the user can weigh the source.

**If the reviewed code touches AI/LLM/agent functionality** (prompt construction, tool calls, embeddings/RAG, agentic loops inside the plugin): also check for prompt injection via user-controlled input reaching a prompt unsanitised, excessive agency (the plugin letting an LLM trigger capability-gated Moodle actions without an explicit user-approval step), and hidden context exposure (sensitive Moodle data — user records, other courses — leaking into a prompt or a third-party API call beyond what the feature needs).

**If the plugin repo defines its own coding standards doc** (a `CONTRIBUTING.md`, a `docs/coding-standards.md`, or similar): read it and treat it as authoritative over the general guidance above where the two conflict.

---

## Phase 3: read the diff or files

1. No files or path given? Run `git diff HEAD` (staged + unstaged) in the current working directory. Empty? Report that there's nothing to review — don't ask further questions.
2. Specific files or a path given? Read those instead of a diff.
3. Large diff (>500 lines)? Split it per file and treat each file as its own review unit, so nothing gets lost to context limits.

---

## Phase 4: evaluate and report

**Sad-path check:** alongside functional correctness, explicitly check whether any new or changed automated test covers a failure path, not just the happy path — invalid input, a missing capability, non-existent data. A new validation, capability check or error-handling branch with no corresponding negative test is worth flagging.

**AI test coverage (only if the diff touches AI/LLM/agent functionality):** beyond the sad-path check above, check whether the change includes at least one adversarial test scenario relevant to the feature — not just a happy-path test with well-behaved input. Depending on what the feature actually does: a prompt injection attempt (user-controlled input, or RAG-retrieved content, trying to override the system prompt), an out-of-scope or jailbreak-style input, an attempt to leak the system prompt or another user's data, or — for agentic code — an attempt to trigger a tool call outside its intended scope. Only flag the scenarios that actually apply to the feature (e.g. don't require a tool-misuse test for a plugin that doesn't call any tools).

Per finding:
- **Severity** — critical / warning / info
- **File:line**
- **What & why** — concrete, not a bare "insecure" or "not compliant"
- **Source** — which Moodle convention, which plugin-specific rule, or "general knowledge" if it isn't a documented Moodle rule
- **Suggestion** — a concrete fix, not a vague recommendation

Report format: markdown in chat, grouped by severity, with a summary count at the top (critical/warning/info).

This skill **never edits code itself**. A fix is an explicit follow-up step ("apply suggestion 2") — never applied automatically, regardless of severity.

---

## Limitations

- Not a replacement for `moodle-plugin-ci` — use that for the authoritative, CI-automatable PHPCS/PHPUnit/Behat check; this skill is a fast conversational pass, better suited to catching things a linter can't (missing sesskey checks, missing Privacy API coverage, questionable API choices).
- Moodle-specific guidance here is written from general Moodle development knowledge, not pulled from a live, versioned source — cross-check against the [official Moodle DevDocs](https://moodledev.io/) for the plugin's specific Moodle version when something looks borderline.
- No memory between reviews — each run is scoped to the diff at that moment.
