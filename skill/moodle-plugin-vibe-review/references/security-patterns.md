# Moodle security patterns for plugin review

Review rules derived from Moodle's published security announcements (MSAs). Each rule is based on the actual fix commit in `moodle/moodle`, not on the advisory text alone. Use it as an extra checklist next to the generic checks in `SKILL.md` Phase 2.

- **Source of truth:** this file is the single place for these rules. Other documents link here instead of copying the rules.
- **Basis:** fix commits on the main branch only; backports to stable branches were not compared.
- **Maintenance:** new MSAs are processed with the quarterly `moodle-coding-standaarden-scan` (advisory → rule). A rule is only added when the fix diff is known.
- **Status labels:** *Confirmed* = visible in the fix diff. *Inferred* = the diff shows the change, but the root cause is our reading; the commit message does not say so.

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
- **Evidence:** MSA-26-0039 / MDL-88335 (Confirmed: `s()` added in two places). MSA-26-0034 / MDL-88981 (Inferred: the label moved from the template into the exporter; the exact unsafe construct was not identified).

### R8. Visibility checks need the right user and context
- **Look at:** code that lists or filters user profile fields or other items with a per-viewer visibility setting.
- **Bad:** loading data with an empty user id, or calling `is_visible()` without a context, so the visibility setting is not evaluated for the viewer.
- **Good:** pass a valid user and an explicit context (e.g. `is_visible(\core\context\system::instance())`).
- **Severity:** medium.
- **Evidence:** MSA-26-0040 / MDL-87518 (Confirmed).

## Seen but not relevant for plugins

- **MSA-26-0043 / MDL-87817:** bypass of the login notification in `complete_user_login()`. Core login logic; plugins do not implement it.

## Adding a rule

1. Take the MSA number, MDL ticket and fix commit from moodle.org/security, the tracker and `moodle/moodle`.
2. Read the diff and describe the bad and good form from it. No diff found: do not write a rule.
3. Check whether an existing rule already covers the class; extend it with the extra evidence instead of adding a near-duplicate.
4. Mark it *Confirmed* or *Inferred*.
