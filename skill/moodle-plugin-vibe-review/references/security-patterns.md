# Moodle security patterns for plugin review

Review rules derived from Moodle's published security announcements (MSAs). Each rule is based on the actual fix commit in `moodle/moodle`, not on the advisory text alone. Use it as an extra checklist next to the generic checks in `SKILL.md` Phase 2.

- **Source of truth:** this file is the single place for these rules. Other documents link here instead of copying the rules.
- **Basis:** fix commits on the main branch only; backports to stable branches were not compared.
- **Maintenance:** new MSAs are processed with the quarterly `moodle-coding-standaarden-scan` (advisory → rule). A rule is only added when the fix diff is known.
- **Status labels:** *Confirmed* = visible in the fix diff. *Inferred* = the diff shows the change, but the root cause is our reading; the commit message does not say so.
- **Second source (F rules):** rules F1-F11 come from a third-party review checklist, not from MSAs. They are labelled *Verified in core* (the core behaviour was read in `moodle/moodle` on 2026-10-09) or *Field report* (the author's review experience only; the rule itself is not proven by core code).

## What goes wrong in practice

In MSA-26-0034 to 26-0043 (Sept 2026) access control dominates: capability checks that are missing or wrong, state that is not respected (plugin disabled, group not in course, field visibility). XSS, CSRF and one blind SQL injection (via an unvalidated field name) complete the picture. Check *which* check is present and *in which context*, not only whether one exists.

## Rules

### R1. The capability must match the action being performed
- **Look at:** `classes/external/*.php`, ajax scripts, any `require_capability()`/`has_capability()`/availability helper call.
- **Bad:** a check that exists but names another action, e.g. an availability helper called with `'generate_text'` inside the *generate image* service.
- **Good:** the action name or capability passed to the check is the one belonging to this exact operation, in the right context (course, module, system).
- **Severity:** high (false assurance; the review sees a check and moves on).
- **Evidence:** MSA-26-0036 / MDL-88587 (Confirmed).

### R2. Access check before any state-changing or recalculating action
- **Look at:** standalone entry points (`view.php`, `index.php`, ajax/external endpoints) that trigger a recalculation, sync, or write when the page is opened.
- **Bad:** the action runs as soon as the page loads, with no check that the user may access the target item.
- **Good:** resolve the target first, verify access (capability and, where relevant, that the item is reachable for this user), throw `moodle_exception` otherwise, then act.
- **Severity:** high.
- **Evidence:** MSA-26-0037 / MDL-88585 (Confirmed: access check added before recalculation; the exact capability used was not inspected).

### R3. Respect the enabled/disabled state of the feature
- **Look at:** entry points of an enrolment, auth, block or other pluggable component.
- **Bad:** only checking that a plugin instance exists (`enrol_get_plugin()`), not that it is enabled. A capability check can be correct and the feature still must not work when disabled.
- **Good:** an explicit enabled-check (e.g. `enrol_is_enabled('manual')`) that throws a `plugindisabled` exception.
- **Severity:** medium.
- **Evidence:** MSA-26-0035 / MDL-88593 (Confirmed).

### R4. Related IDs from the request must belong to the parent object
- **Look at:** ajax/external/form handlers that take a second id next to the main one (group id with course id, cohort, section, category).
- **Bad:** using `required_param('group', ...)` directly in an action such as adding a user to a group.
- **Good:** fetch the record with the parent in the condition: `$DB->get_record('groups', ['id' => $groupid, 'courseid' => $course->id])`. Act only when it exists. Keep the capability check on top of it.
- **Severity:** medium.
- **Evidence:** MSA-26-0038 / MDL-88538 (Confirmed).
- **Also:** authorise from the id in the URL context, never from an id submitted in the body; re-check the capability where a stored id is acted on later (see F4).

### R5. Field and column names from stored configuration are whitelisted before use
- **Look at:** conditions, report builders, filters, anywhere a stored setting (a profile field name, a column name) ends up in a query or in a lookup of user data.
- **Bad:** using the stored name as-is. Stored data is not trusted input; it can be changed or crafted.
- **Good:** `array_key_exists($field, <list of allowed fields>)` first, `return false` for an unknown field. Add a unit test with an unknown field.
- **Severity:** high (SQL injection class).
- **Evidence:** MSA-26-0042 / MDL-89484 (Confirmed: whitelist check added; the SQL part of the injection itself was not seen in the diff).

### R6. Standalone import/processing scripts need request protection
- **Look at:** `import.php`-style scripts that fetch and commit data and are reached by a redirect from another page.
- **Bad:** a second entry point that processes data directly, outside the sesskey-protected form flow.
- **Good:** process within the protected form flow (`require_sesskey()` / form `get_data()`), or restrict the script to its one legitimate caller and throw otherwise.
- **Severity:** high.
- **Evidence:** MSA-26-0041 / MDL-84545 (Inferred: the commit says "Improve validation in XML grade import"; the CSRF reading comes from the diff and the advisory title).

### R7. Escape user-controlled values in emails and templates
- **Look at:** strings assembled for emails and notifications (`$data->username = $user->username`), and `aria-label`/attribute values built inside Mustache templates.
- **Bad:** putting a user-controlled value (username, subject, display name) into HTML-bearing strings without escaping.
- **Good:** `s($user->username)` for values that go into an HTML message; compute labels in the PHP exporter (typed `PARAM_TEXT`) and print them as `{{label}}` instead of composing them in the template.
- **Severity:** medium.
- **Also:** `html_writer` contents are not escaped; see F2.
- **Evidence:** MSA-26-0039 / MDL-88335 (Confirmed: `s()` added in two places). MSA-26-0034 / MDL-88981 (Inferred: the label moved from the template into the exporter; the exact unsafe construct was not identified).

### R8. Visibility checks need the right user and context
- **Look at:** code that lists or filters user profile fields or other items with a per-viewer visibility setting.
- **Bad:** loading data with an empty user id, or calling `is_visible()` without a context, so the visibility setting is not evaluated for the viewer.
- **Good:** pass a valid user and an explicit context (e.g. `is_visible(\core\context\system::instance())`).
- **Severity:** medium.
- **Evidence:** MSA-26-0040 / MDL-87518 (Confirmed).

## Rules from release reviews (third-party field reports)

Source: the `moodle-release-preflight` and `moodle-field-lessons` skills in [adamjenkins/claude-moodle-dev](https://github.com/adamjenkins/claude-moodle-dev/tree/wisecat/moodle-5.3) (MIT, branch `wisecat/moodle-5.3`, v0.6.0). The rules are rewritten, not copied. Each rule was checked against core by one reviewer and then by an independent second reviewer who tried to refute it. Cite the rule ID (e.g. F1) in findings.

### F1. Student-uploaded files are not served inline unless they are safe media
- **Look at:** `send_stored_file(` calls in `pluginfile` callbacks and `lib.php` (4th argument `$forcedownload`).
- **Bad:** `$forcedownload = false` for HTML or other active content that a student uploaded; it renders inline in the grader's session (stored XSS).
- **Good:** force the download unless the file is an image, audio or video type. Core derives the sent MIME type from the file extension, so allowlist the extension when storing, not just `PARAM_FILE` on the name. Allowlist `$filearea` in the callback.
- **Severity:** high.
- **Evidence:** *Verified in core:* `send_file()` forces download only for SVG (`lib/filelib.php`, comment cites SVG XSS) and sets no CSP or `sandbox` for other types. The XSS consequence for HTML is *Field report*.

### F2. Escape `html_writer` contents (extends R7)
- **Look at:** `html_writer::tag()`, `::div()`, `::span()`, `::link()` calls, and JS sinks (`innerHTML`, `insertAdjacentHTML`).
- **Bad:** a stored or user value as the *contents* argument. Attributes are escaped; contents are not.
- **Good:** `s()`, `format_string()` or `format_text()` at the point of output. For JS sinks, trace the feed: HTML from `Templates.renderForPromise()` using `{{ }}` is fine; string-concatenated HTML is not. A grep hit alone is not a finding.
- **Severity:** medium.
- **Evidence:** *Verified in core:* `html_writer::tag()` returns start tag + `$contents` + end tag unescaped (`lib/classes/output/html_writer.php`).

### F3. Restored structural values are validated like form input
- **Look at:** `process_*()` methods in `backup/moodle2/restore_*`.
- **Bad:** a restored value used as a field name, key, enum, id or other structural value without the validation the interactive form applies.
- **Good:** the same `clean_param()` or allowlist as the form, before `insert_record()`/`update_record()`; upsert when a unique index exists. Free text may stay as restored if it is cleaned on output.
- **Severity:** medium for structural values, low for free text.
- **Evidence:** *Verified in core:* the restore base steps and core restore steps insert restored data without cleaning and rely on output cleaning (`backup/moodle2/restore_activity_structure_step.class.php`, `mod/assign/backup/moodle2/restore_assign_stepslib.php`). The rule itself is a *Field report*.

### F4. Re-check the capability where a stored value is acted on (extends R1/R2)
- **Look at:** adhoc or scheduled tasks and external functions that use an id stored earlier.
- **Bad:** authorising only at request time; a tampered stored value runs later with task privileges.
- **Good:** re-check the capability and re-scope the id inside the task or service.
- **Severity:** high.
- **Evidence:** *Field report.*

### F5. Limits are enforced on the server
- **Look at:** size, duration, count or rate limits that exist only in `amd/src/*.js`.
- **Bad:** a direct POST ignores them.
- **Good:** mirror each limit server-side (`get_user_max_upload_file_size()`, a `$DB->count_records()` gate).
- **Severity:** medium.
- **Evidence:** *Field report.*

### F6. Two roles do not write the same state value
- **Look at:** status, enum or flag columns that more than one capability level writes.
- **Bad:** an owner-level toggle reuses the value a moderator uses for a takedown, so the owner silently undoes it.
- **Good:** a distinct value per actor (or store the actor); the lower role's setter refuses to touch the higher role's value.
- **Severity:** low to medium (an integrity flaw rather than a vulnerability).
- **Evidence:** *Field report.*

### F7. Question engine: filter the slots yourself
- **Look at:** `process_all_actions()` called with request data or with `$postdata` that has no `slots` key.
- **Bad:** the client picks or omits the slots; without a `slots` key every slot in the usage is processed.
- **Good:** keep only the allowed slots' `get_field_prefix($slot)` keys and set `slots` yourself. A regression test needs `:sequencecheck` plus a hostile `-submit`, or it passes vacuously.
- **Severity:** medium (high where slots are gated by page or time).
- **Evidence:** *Verified in core:* `question/engine/questionusage.php`, `get_slots_in_request()` returns `get_slots()` when `slots` is absent.

### F8. Activity modules implement the full course-reset set
- **Look at:** `mod_*` plugins that store user data.
- **Bad:** only `<mod>_reset_userdata()`. It still runs, but without `<mod>_reset_course_form_definition()` the reset form has no option for the plugin, and core shows no "unsupported" notice.
- **Good:** implement `_reset_course_form_definition()`, `_reset_course_form_defaults()` and `_reset_userdata()`, including the gradebook.
- **Severity:** medium.
- **Evidence:** *Verified in core:* `course/reset_form.php` and `lib/moodlelib.php` (`reset_course_userdata()`).

### F9. Time zones: pass the user object, not the raw field
- **Look at:** `core_date::get_user_timezone()` calls with another user's data.
- **Bad:** passing the raw `$user->timezone` value `99`; the bare value `99` resolves to the *viewer's* zone (`$USER`).
- **Good:** pass the user object (an object with `99` returns the server zone), or call `core_date::get_server_timezone()`. `$CFG->forcetimezone` overrides all of this.
- **Severity:** low to medium (a correctness rule more than a security rule).
- **Evidence:** *Verified in core:* `lib/classes/date.php`, `get_user_timezone()`.

### F10. Privacy provider: everything declared is exported and deleted (extends the Privacy API check)
- **Look at:** `classes/privacy/provider.php` against `get_metadata()` and `db/install.xml`.
- **Bad:** a declared table or field that no export or delete path handles; a renamed table still queried by its old name.
- **Good:** each declared item appears in export and in every delete method; `tests/privacy/provider_test.php` uses generated rows.
- **Severity:** medium.
- **Evidence:** *Field report.*

### F11. A green run over an empty `tests/` is not coverage (extends the sad-path check)
- **Look at:** `tests/` against the paths under review (upload, capability, privacy).
- **Bad:** `moodle-plugin-ci phpunit`/`behat` pass because no tests exist; a security test that was never seen failing.
- **Good:** list the tests that cover each sink, and run each security test once against the unfixed code.
- **Severity:** low (process); state it in the report.
- **Evidence:** *Field report.*

## Seen but not relevant for plugins

- **MSA-26-0043 / MDL-87817:** bypass of the login notification in `complete_user_login()`. Core login logic; plugins do not implement it.

## Adding a rule

1. Take the MSA number, MDL ticket and fix commit from moodle.org/security, the tracker and `moodle/moodle`.
2. Read the diff and describe the bad and good form from it. No diff found: do not write a rule.
3. Check whether an existing rule already covers the class; extend it with the extra evidence instead of adding a near-duplicate.
4. Mark it *Confirmed* or *Inferred*.
