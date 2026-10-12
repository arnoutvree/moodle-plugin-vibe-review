# Recurring defects in production Moodle plugins

Review rules derived from defects that were found and fixed in the commit history of about 65 plugins developed in-house. Use it as an extra checklist next to `security-patterns.md` and the generic checks in `SKILL.md` Phase 2. Cite the rule ID (e.g. D12) in findings.

- **Source:** recurring patterns in commits, merge requests and issues of real plugins (2026). No project names or links are included.
- **Evidence label:** every rule is a *Field report*: it was seen in practice, but it was not verified against Moodle core code. The count says in how many different plugins the defect occurred.
- **Maintenance:** grouping and wording were produced with local language models and have not been checked rule by rule against Moodle core; check borderline cases against the [Moodle DevDocs](https://moodledev.io/) for the plugin's Moodle version.

## Rules

### Access control and authorisation

#### D1. Authentication and Capability Checks in PHP Files
- **Look at:** All public-facing PHP entry points and scripts.
- **Bad:** Executing database queries or rendering content immediately after loading config.php without calling require_login() or checking user capabilities.
- **Good:** Calling require_login() and using require_capability() or is_capable() to verify permissions before processing data or displaying protected content.
- **Severity:** high
- **Evidence:** *Field report*, seen in 4 plugins.

#### D2. Correct Use of Contexts and Capabilities
- **Look at:** Permission checks, capability validations, and role-based access control logic.
- **Bad:** Using hardcoded role IDs (e.g., $USER->role == 2) or using context_system::instance() for checks that should be specific to a course, module, or user.
- **Good:** Using has_capability() with the most specific context available (e.g., $context = context_module::instance($cmid)) and avoiding direct role ID comparisons.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D3. Authorization for Group and User Data Access
- **Look at:** Code accessing group members via get_users() or assigning $USER->id to record ownership fields in database queries.
- **Bad:** Calling get_users() for all groups or inserting $USER->id as an owner without verifying required capabilities like moodle/site:accessallgroups.
- **Good:** Explicitly checking required capabilities using require_capability() or has_capability() before granting access to all groups or assigning ownership.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D4. Client-side and Session Security
- **Look at:** AMD modules, JavaScript files, and PHP controllers handling session data or UI visibility.
- **Bad:** Showing UI elements based on client-side logic alone or trusting $SESSION data without verifying it against the current $USER->id.
- **Good:** Performing server-side capability checks before rendering UI and validating that session-stored identifiers match the authenticated $USER->id.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D5. CSRF Protection and Form Validation
- **Look at:** Publicly accessible PHP scripts and controllers processing POST requests or updating data.
- **Bad:** Updating database records or performing actions using $_POST data without verifying the session key.
- **Good:** Calling require_sesskey() or using Moodle's form API to ensure requests are legitimate and protected against CSRF.
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D6. Capability Definition and Consistency
- **Look at:** db/access.php and calls to require_capability() or has_capability() throughout the codebase.
- **Bad:** Capabilities are missing CAP_PREVENT for guest/user roles, or the capability name used in the code differs from the one defined in db/access.php.
- **Good:** All capabilities are explicitly defined in db/access.php with correct contexts and are referenced using the exact same string in all capability checks.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 6 plugins.

#### D7. Separate Resource Registration from Access Control
- **Look at:** Administration menu registrations, such as calls to admin_externalpage_add(), within setup or initialization functions.
- **Bad:** Wrapping the registration function itself inside a conditional access check: if ($hassiteconfig) { admin_externalpage_add(...); }
- **Good:** Register the resource unconditionally; handle access control within the target page or via the API's built-in permission parameters.
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

### Input, output and data validation

#### D8. Validate Object Properties and Records Before Access
- **Look at:** Direct access to object properties, array elements, or database record fields, and method calls on returned objects.
- **Bad:** $id = $record->id; // Accessing property without verifying if $record is an object or not null
- **Good:** if ($record && isset($record->id)) { $id = $record->id; }
- **Severity:** high
- **Evidence:** *Field report*, seen in 8 plugins.

#### D9. Validate Array Types Before Iteration
- **Look at:** Loops (foreach) and array functions acting on input variables or dynamic data sources.
- **Bad:** foreach ($input as $item) { ... } // Used without verifying if $input is actually an array.
- **Good:** if (is_array($input)) { foreach ($input as $item) { ... } }
- **Severity:** high
- **Evidence:** *Field report*, seen in 3 plugins.

#### D10. Validate Data Before Database Write Operations
- **Look at:** Calls to $DB->insert_record(), $DB->update_record(), or $DB->delete_records() where input data or identifiers are used.
- **Bad:** $DB->update_record('table', $data); // Updating without verifying if $data contains required fields or valid identifiers.
- **Good:** if (!empty($data['id']) && validate_params($data, $params)) { $DB->update_record('table', $data); }
- **Severity:** high
- **Evidence:** *Field report*, seen in 3 plugins.

#### D11. Use Moodle Parameter API for Input Validation
- **Look at:** Code handling URL parameters, request data, or external input from $_GET, $_POST, and $_SERVER.
- **Bad:** Directly accessing superglobals like $_GET['id'] or using manual string manipulation to extract and validate parameters.
- **Good:** Using Moodle's parameter API functions such as optional_param() or required_param() to retrieve and validate input.
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D12. Sanitize Whitespace and Delimiters in Dynamic Input
- **Look at:** Code handling dynamic user input used for filenames, usernames, or identifiers before storage or filesystem operations.
- **Bad:** Directly concatenating raw user input into filenames or identifiers without stripping whitespace, line breaks (\n, \r), or illegal delimiters.
- **Good:** Using functions like clean_param() or regex to remove whitespace and control characters before using input in filenames or system identifiers.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.

#### D13. Strict Type Comparison and Casting
- **Look at:** Conditional statements and comparisons involving database records, API responses, or mixed-type variables.
- **Bad:** if ($row->id == $some_int) // Comparing a potential string from the database with an integer using loose comparison
- **Good:** if ((int)$row->id === $some_int) // Explicitly casting to integer and using strict comparison
- **Severity:** medium
- **Evidence:** *Field report*, seen in 2 plugins.

#### D14. Sanitize Input and Filter Expired Records
- **Look at:** Input handling logic and SQL queries using get_records_sql or similar database API methods.
- **Bad:** Directly echoing raw variables without trim() or fetching records without including a WHERE clause to filter out expired dates.
- **Good:** Using trim() on dynamic input and adding time-based constraints (e.g., 'expiry > :now') directly within the SQL query.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D15. Correct Moodle PARAM Types
- **Look at:** Calls to required_param(), optional_param(), and param_text() throughout the plugin.
- **Bad:** Using PARAM_ALPHANUM for fields that should allow spaces, punctuation, or free-form text (e.g., required_param('description', PARAM_ALPHANUM);).
- **Good:** Using PARAM_TEXT for general text or PARAM_RAW for content requiring minimal filtering (e.g., required_param('description', PARAM_TEXT);).
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D16. Consistent Mapping of External Data to Moodle Fields
- **Look at:** Data transformation logic and mapping functions that convert external API responses or imported data into Moodle database records.
- **Bad:** Mapping logic missing fallback values (else branches) or inconsistent field renaming during transformation, leading to null or unpredictable values in Moodle fields.
- **Good:** Explicit mapping for every field with defined default values for missing external data to ensure database integrity.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D17. Pass Boolean User Field Values as Strings
- **Look at:** Calls to Moodle user API functions that update custom profile fields or status flags.
- **Bad:** Passing boolean or integer values (0 or 1) directly to the API: update_user_status($id, 1);
- **Good:** Passing boolean values as strings ('0' or '1') to ensure API compatibility: update_user_status($id, '1');
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D18. Validate json_decode return type for array declarations
- **Look at:** Methods with an ': array' return type hint that call json_decode().
- **Bad:** return json_decode($json); // Returns an object by default, causing a TypeError if the method expects an array.
- **Good:** return json_decode($json, true); // The second parameter ensures an associative array is returned.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

### Database and SQL

#### D19. SQL Injection and Parameter Safety
- **Look at:** All $DB API method calls, specifically those using custom SQL strings like get_records_sql() or execute().
- **Bad:** $DB->get_records_sql("SELECT * FROM {user} WHERE username = '" . $username . "'");
- **Good:** $DB->get_records_sql("SELECT * FROM {user} WHERE username = :username", ['username' => $username]);
- **Severity:** high
- **Evidence:** *Field report*, seen in 8 plugins.

#### D20. SQL Filtering and Access Control
- **Look at:** Custom SQL queries in db/install.xml, local libraries, or repository classes targeting user, role, or context tables.
- **Bad:** SELECT count(*) FROM {user} WHERE courseid = :cid (missing checks for deleted or suspended status).
- **Good:** SELECT count(*) FROM {user} WHERE courseid = :cid AND deleted = 0 AND suspended = 0
- **Severity:** high
- **Evidence:** *Field report*, seen in 6 plugins.

#### D21. Data Integrity During Updates and Insertions
- **Look at:** Database interaction logic using the $DB API, specifically insert_record and update_user calls.
- **Bad:** Ignoring return values from $DB->insert_record(), manually incrementing IDs (e.g., $id + 1), or calling update_user() without first merging data into the existing user object.
- **Good:** Assigning $DB->insert_record() results to a variable, relying on database auto-increment, and using the Moodle API to merge and update records correctly.
- **Severity:** high
- **Evidence:** *Field report*, seen in 4 plugins.

#### D22. Database Schema and XMLDB Consistency
- **Look at:** install.xml, db/upgrade.php, and any code performing database queries or using flexible tables (sortable/editable).
- **Bad:** Hardcoded table names without prefixes or using column names in code (e.g., in sortable()) that do not match the definitions in install.xml.
- **Good:** Table and column names in the code exactly match the XMLDB definitions and utilize the correct plugin prefix via the Moodle API.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 5 plugins.

#### D23. Query Optimization and JOIN Logic
- **Look at:** Database queries using $DB methods, specifically those involving JOINs or executed within loops.
- **Bad:** Using ambiguous column names without aliases, incorrect JOIN types, or performing heavy operations like LIKE inside loops.
- **Good:** Using explicit table aliases, appropriate JOIN types for the logic, and prefetching data to avoid queries inside loops.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 4 plugins.

#### D24. Deletion Logic and Recursion
- **Look at:** Database interaction logic, specifically within functions handling record removal, updates, or hierarchical data structures.
- **Bad:** Using delete-insert cycles to update records, relying solely on return values for deletion confirmation, or failing to handle recursive child records.
- **Good:** Use $DB->update_record() for modifications, verify database state after deletions, and implement recursive functions for hierarchical data cleanup.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 2 plugins.

#### D25. SQL Syntax and Result Validation
- **Look at:** Database queries within data managers or local API calls.
- **Bad:** SELECT firstname AS 'firstname' FROM {user} (using quotes for aliases or omitting ORDER BY and required fields for fullname()).
- **Good:** SELECT firstname AS firstname FROM {user} ORDER BY firstname ASC (using unquoted aliases and ensuring all fields required by Moodle APIs are selected).
- **Severity:** low
- **Evidence:** *Field report*, seen in 4 plugins.

### Privacy and personal data

#### D26. Implement Moodle Privacy API Provider
- **Look at:** classes/privacy/provider.php
- **Bad:** Missing provider.php, using null_provider despite storing PII, or implementing privacy_provider without required methods like export_user_data() and delete_user_data().
- **Good:** A complete privacy provider implementing privacy_provider interface that accurately exports and deletes all PII data according to core API specifications.
- **Severity:** high
- **Evidence:** *Field report*, seen in 11 plugins.

#### D27. Correct Use of Email Fields
- **Look at:** User-related functions and data mapping where values are assigned to the $user->email field or the user table email column.
- **Bad:** $user->email = $internalId; // Assigning internal identifiers or sensitive IDs to the email field
- **Good:** $user->email = $emailAddress; // Assigning only valid email addresses to the email field
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D28. Prevent PII in Global JavaScript Configuration
- **Look at:** Calls to M.cfg within before_standard_html_head() or AMD modules that assign user data to the global configuration object.
- **Bad:** M.cfg.userName = $USER->firstname;
- **Good:** Pass data to AMD modules via page.require.js or use a secure, scoped data attribute in the HTML DOM.
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D29. GDPR Privacy Provider and Capability Definitions
- **Look at:** db/access.php and the presence of a privacy provider class implementing privacy_provider
- **Bad:** Missing a privacy provider for GDPR compliance or missing required capabilities in db/access.php, leading to incorrect permission checks or data privacy violations.
- **Good:** A privacy provider is implemented to handle data requests and all necessary capabilities are defined in db/access.php with the appropriate context.
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D30. Filter Non-Student Roles in User Statistics Aggregations
- **Look at:** SQL queries performing aggregations (COUNT, SUM, AVG) on user-related tables for student reporting.
- **Bad:** Queries that aggregate data across all users in a course without filtering by specific student roles, potentially including teachers or administrators.
- **Good:** Queries that join with the role tables or use Moodle API functions to ensure only users with the student role are included in the aggregation.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

### API currency and compatibility

#### D31. Use Context Class Factory Methods
- **Look at:** Calls to context retrieval functions throughout the plugin logic.
- **Bad:** Using global functions like get_context_instance(CONTEXT_SYSTEM) or get_courseid_from_context($context).
- **Good:** Using static factory methods from specific context classes, such as context_system::instance() or context_course::instance($courseid).
- **Severity:** high
- **Evidence:** *Field report*, seen in 6 plugins.

#### D32. Modern Event API and Context Usage
- **Look at:** Event triggers in PHP code and db/events.php definitions.
- **Bad:** Using events_trigger(), using $handlers in db/events.php, or calling event::create() without a valid 'context' key.
- **Good:** Using the core\event API with event::create() including a valid context and defining $observers in db/events.php.
- **Severity:** high
- **Evidence:** *Field report*, seen in 3 plugins.

#### D33. Version.php and Upgrade Consistency
- **Look at:** version.php and db/upgrade.php
- **Bad:** $plugin->version = 20231010; // Incorrect format or mismatch between $plugin->version and upgrade_plugin_savepoint() in db/upgrade.php
- **Good:** $plugin->version = 2023101000; // 10-digit YYYYMMDDxx format synchronized with the version check in db/upgrade.php
- **Severity:** high
- **Evidence:** *Field report*, seen in 3 plugins.

#### D34. Event Registration and Observer Validation
- **Look at:** db/events.php and event observers
- **Bad:** Missing core event definitions in db/events.php or calling redirect() within an observer without first validating the current state or conditions in the database.
- **Good:** All custom events registered in db/events.php and observers performing strict database validation before triggering any redirects or state changes.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D35. API Currency and Type Declarations
- **Look at:** Method signatures, return statements, and conditional comparisons throughout the PHP codebase.
- **Bad:** Using deprecated Moodle APIs, missing return types, or loose comparisons like if ($var == "").
- **Good:** Using current Moodle APIs, strict type declarations (e.g., : void, : int), and strict comparisons (===).
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.

#### D36. Method Signatures of Overridden Functions
- **Look at:** Overridden methods in classes extending Moodle core or plugin base classes.
- **Bad:** public function instance_config_save() { // Missing parameters defined in the parent class signature }
- **Good:** public function instance_config_save($params) { // Signature exactly matches the parent class definition }
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D37. Consistent Event Naming and Enrollment Status Values
- **Look at:** Event class definitions and enrollment status assignments within the plugin logic.
- **Bad:** Using generic event class names that may conflict with other plugins or using non-standard integers for enrollment status (e.g., using 1 instead of 0 for active).
- **Good:** Using unique, prefix-based event class names and adhering to Moodle's core enrollment status constants and standard values.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D38. Plugin Version Requirements
- **Look at:** The version.php file of the plugin.
- **Bad:** $plugin->requires = 2022112800; // Set to a version lower than the actual API features used in the code.
- **Good:** $plugin->requires = 2023100900; // Set to the lowest Moodle version that actually supports all used APIs.
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

### Logic, dates and error handling

#### D39. Secure Redirects and Input Validation
- **Look at:** Redirect logic and URL construction within controller or page scripts.
- **Bad:** Manually concatenating $CFG->wwwroot to build redirect URLs or using unvalidated input parameters directly in redirect functions.
- **Good:** Using Moodle core helper functions like redirect() with relative paths or validated parameters to prevent open redirect vulnerabilities.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D40. Consistency of Status and Dependent Subsystems
- **Look at:** Database update queries, state transition logic, and reset functions within managers or API classes.
- **Bad:** Updating a status field while neglecting dependent fields, failing to call cleanup functions like grade_user_unenrol(), or incomplete initialization of state variables.
- **Good:** Ensuring all related fields and subsystems are updated atomically or reset to default values whenever a primary state change occurs.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D41. Consistency of Logic and Boundary Values
- **Look at:** Conditional statements, numerical thresholds, and accompanying developer comments within PHP logic.
- **Bad:** Contradictions between comments and operators, or overlapping thresholds: if ($val > 10) { ... } followed by if ($val >= 10) { ... }
- **Good:** Mutually exclusive conditions with matching comments: if ($val > 10) { ... } else if ($val <= 10) { ... }
- **Severity:** medium
- **Evidence:** *Field report*, seen in 4 plugins.

#### D42. Correct Date and Timestamp Handling
- **Look at:** Date input validation, database field processing, and timestamp calculations using functions like strtotime().
- **Bad:** Passing database fields directly to strtotime() without null checks, performing redundant date conversions, or passing objects to setTimestamp().
- **Good:** Validate date inputs for null values, use raw source data for timestamps, and calculate new times based on the current system time.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 4 plugins.

#### D43. Safe Loop Variable Management and Scope
- **Look at:** Variable assignments and data aggregation within foreach and while loops.
- **Bad:** Overwriting a variable with '=' inside a loop or mutating object properties without resetting them between iterations, causing data loss or leakage.
- **Good:** Using concatenation, array_push(), or array_merge() to aggregate results, and ensuring loop-specific variables are initialized or reset at the start of each iteration.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.

#### D44. Completion Actions and Notification Timing
- **Look at:** Database transaction blocks and logic handling activity completion or user notifications.
- **Bad:** Calling email_to_user() or updating completion status before $DB->transactionCommit() is executed.
- **Good:** Performing all database updates first and triggering notifications or completion markers only after a successful transaction commit.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 2 plugins.

#### D45. Multi-level Status Validation
- **Look at:** Business logic classes and service layers handling state transitions or permission checks.
- **Bad:** Validating a status change at only one level (e.g., checking user status while ignoring the parent course or system-wide status).
- **Good:** Implementing hierarchical status checks that verify both individual entity states and global/parent context before executing business logic.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

### Frontend: forms, templates, JavaScript, strings

#### D46. Language Files and i18n Validation
- **Look at:** Language PHP files (lang/en/...) and get_string() calls throughout the codebase.
- **Bad:** Missing keys in language files, using non-standard key names, or including special typographic characters like em dashes (—) instead of standard hyphens.
- **Good:** All get_string() keys defined in lang files, using standard naming conventions (e.g., 'pluginname'), and using standard ASCII characters for compatibility.
- **Severity:** high
- **Evidence:** *Field report*, seen in 7 plugins.

#### D47. Consistent Form Definition and Data Mapping
- **Look at:** The Moodle form class, specifically comparing the definition() method with the set_data() call in the form's instantiation or usage.
- **Bad:** Adding a new element to definition() but failing to include the corresponding field in the array passed to set_data(), leaving the field empty on load.
- **Good:** Ensuring every field defined in definition() is correctly mapped and provided in the data array passed to set_data().
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D48. Use optional_param for ID Validation in Forms
- **Look at:** The definition_after_data() method within Moodle form classes.
- **Bad:** Using required_param('id', PARAM_INT) without checking if the request is for a file draft area, which causes crashes during file uploads.
- **Good:** Use optional_param('id', 0, PARAM_INT) or wrap required_param in a check for file_prepare_draft_area().
- **Severity:** high
- **Evidence:** *Field report*, seen in 1 plugin.

#### D49. UI Visibility and Conditional Logic
- **Look at:** Output rendering logic, AMD modules, and templates where UI elements are generated based on data availability or action results.
- **Bad:** Calling html_writer::table($data) or .show() without first verifying that $data is not empty or that the preceding promise resolved successfully.
- **Good:** Wrapping UI generation in if (!empty($data)) checks or using promise .then() blocks to ensure elements only appear when relevant data exists.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 4 plugins.

#### D50. URL Generation and Page Settings
- **Look at:** PHP files handling page requests and link generation.
- **Bad:** Hardcoded paths, manual concatenation using $CFG->wwwroot, or missing $PAGE->set_url() calls at the start of the page.
- **Good:** Use the moodle_url class for all links and call $PAGE->set_url(new moodle_url('/path/to/page.php')) at the beginning of each page.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.

#### D51. Mustache Context and Data Preparation
- **Look at:** The extend_template_context() method and .mustache template files.
- **Bad:** Calculating percentages or concatenating units (e.g., 'width' => $val . '%') in PHP, or performing logic/calculations directly within Mustache templates.
- **Good:** Prepare final, ready-to-render values in PHP. Ensure all variables passed to the template context have defined default values to avoid undefined property notices.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.

#### D52. CSS Class Validation for Interactive Elements
- **Look at:** JavaScript event handlers and conditional logic checking for element states via CSS classes.
- **Bad:** Using .hasClass() on a target element when the state class is actually applied to a parent container or using incomplete selectors.
- **Good:** Using .closest('.class-name').length or .closest('.class-name').hasClass('active') to ensure the state is correctly detected regardless of nesting.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D53. Consistent jQuery and DOM Manipulation
- **Look at:** JavaScript files and AMD modules performing DOM manipulation.
- **Bad:** Mixing native DOM methods with jQuery chains, such as: $el.addClass('active').removeAttribute('disabled');
- **Good:** Using consistent jQuery methods throughout the chain: $el.addClass('active').removeAttr('disabled');
- **Severity:** low
- **Evidence:** *Field report*, seen in 2 plugins.

#### D54. CSS Management and Styling
- **Look at:** PHP files and output templates.
- **Bad:** Inline <style> blocks echoed directly within PHP files or embedded in HTML output.
- **Good:** CSS placed in separate stylesheets and loaded using $PAGE->requires->css().
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

#### D55. Avoid Redeclaring Variables in Block Scopes
- **Look at:** JavaScript files and AMD modules, specifically within functions containing multiple conditional or loop blocks.
- **Bad:** if (x) { let a = 1; } if (y) { let a = 2; }
- **Good:** let a; if (x) { a = 1; } if (y) { a = 2; }
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

#### D56. Preserve Filter Parameters During Pagination
- **Look at:** URL generation in renderer.php or form handling logic, specifically where pagination or 'per page' links are constructed.
- **Bad:** new moodle_url('/page.php', ['page' => 2]); // Only the page number is passed, discarding existing filter parameters from the current request.
- **Good:** new moodle_url('/page.php', array_merge(get_url_parameters(), ['page' => 2])); // Merges current URL parameters to preserve active filters.
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

#### D57. Use image_url() instead of deprecated pix_url()
- **Look at:** PHP files utilizing the $OUTPUT object for rendering images.
- **Bad:** $OUTPUT->pix_url('/plugin/icon.png');
- **Good:** $OUTPUT->image_url('/plugin/icon.png');
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

#### D58. Progress Bar Parameter Order
- **Look at:** Instantiations of the progressbar class.
- **Bad:** new progressbar($max, $current);
- **Good:** new progressbar($current, $max);
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

### Settings, files and integrations

#### D59. File Management and Moodle File API
- **Look at:** Code handling file storage, directory creation, and path construction within the moodledata directory.
- **Bad:** Using native PHP functions like file_put_contents() or mkdir() with manual path concatenation using $CFG->dataroot.
- **Good:** Using the Moodle File API (e.g., stored_file class) or ensuring directories are recursively created and paths are handled consistently via API methods.
- **Severity:** high
- **Evidence:** *Field report*, seen in 3 plugins.

#### D60. Configuration Management and Hardcoding
- **Look at:** PHP files containing configuration values, email addresses, or user IDs.
- **Bad:** Hardcoding configuration values, emails, or IDs directly in the source code (e.g., $email = 'admin@example.com';).
- **Good:** Retrieving configuration values using the get_config() function to allow administrative overrides via the UI.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D61. External APIs and Access Control
- **Look at:** API call implementations, configuration handling, and page access control logic.
- **Bad:** Using file_get_contents() for URLs, hardcoding API keys, or using require_login() without verifying specific capabilities and the correct context.
- **Good:** Using the Moodle curl class, storing keys in plugin settings, and implementing require_capability() with the appropriate context.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D62. Explicit Admin Setting Default Values
- **Look at:** Admin setting definitions within settings.php files.
- **Bad:** Using hardcoded indices for defaults (e.g., 'default' => $options[0]) or declaring empty strings as defaults (e.g., 'default' => "").
- **Good:** Explicitly defining the default value using a specific constant, a literal value, or omitting the default parameter if the system default is sufficient.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 2 plugins.

#### D63. Use moodle_phpmailer wrapper for sending emails
- **Look at:** Email sending logic and class instantiations.
- **Bad:** Directly instantiating PHPMailer via 'new PHPMailer()' or manually requiring 'phpmailer/class.phpmailer.php'.
- **Good:** Using the 'moodle_phpmailer()' wrapper function to obtain a configured PHPMailer instance.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D64. Ensure Cache Consistency After Data Mutation
- **Look at:** Data mutation methods (update, delete, or create) that affect cached information.
- **Bad:** Updating or deleting a record in the database without subsequently purging the associated cache.
- **Good:** Calling cache::make('\\cache_name')->purge() or a specific cache invalidation method immediately after the data mutation.
- **Severity:** medium
- **Evidence:** *Field report*, seen in 1 plugin.

#### D65. Safe Access to Global Configuration $CFG
- **Look at:** Any direct access to properties of the global $CFG object.
- **Bad:** if ($CFG->my_plugin_setting) { ... }
- **Good:** if ($CFG->my_plugin_setting ?? false) { ... }
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

#### D66. Avoid Hardcoded Conditionals in Scheduled Tasks
- **Look at:** The execute() method within classes implementing \moodle_plugin_task or scheduled task scripts.
- **Bad:** if (true) { // logic } or commented-out conditional checks that force execution regardless of state.
- **Good:** if ($condition) { // logic } where $condition is a dynamic check based on configuration or system state.
- **Severity:** low
- **Evidence:** *Field report*, seen in 1 plugin.

### Code hygiene and tests

#### D67. Autoloading, Namespaces, and Class Structure
- **Look at:** Class definitions, namespace declarations, @package tags, and exception handling in namespaced files.
- **Bad:** Manual require_once calls, incorrect namespace prefixes, @package mismatching the plugin directory, or 'throw new Exception' without a leading backslash in a namespaced class.
- **Good:** Use Moodle's autoloader, correct core namespaces, matching @package declarations, and global exceptions via '\Exception' in namespaced files.
- **Severity:** high
- **Evidence:** *Field report*, seen in 6 plugins.

#### D68. Remove Debug Code and Temporary Output
- **Look at:** All files in the diff, specifically PHP and JavaScript source code.
- **Bad:** Presence of var_dump(), print_r(), die(), exit(), console.log(), console.error(), or git merge markers (<<<<<<<, =======, >>>>>>>).
- **Good:** Production-ready code with no temporary debugging statements; use Moodle's debugging framework (debugging()) for legitimate logs.
- **Severity:** high
- **Evidence:** *Field report*, seen in 4 plugins.

#### D69. Architecture, Data Model, and API Integration
- **Look at:** Database schema definitions, technical specifications, and the top of PHP files for required library imports.
- **Bad:** Using Moodle API functions (e.g., grade_write_grade()) without explicitly loading the necessary library file via require_once().
- **Good:** Ensuring the data model matches specifications and explicitly loading required libraries (e.g., require_once('gradelib.php');) before calling their functions.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D70. Class Implementation and Testing Strategy
- **Look at:** Plugin class constructors and PHPUnit/Behat test suites.
- **Bad:** Constructors with mandatory parameters lacking default values, or using Behat scenarios to test capability-based access restrictions.
- **Good:** Constructors using default parameters for flexibility and using PHPUnit for testing capability assertions.
- **Severity:** high
- **Evidence:** *Field report*, seen in 2 plugins.

#### D71. Syntax, Typing, and Consistency
- **Look at:** PHP files, specifically object property access, array keys, and function arguments compared to the plugin documentation.
- **Bad:** $USER->firstnme = 'Jan';
- **Good:** $USER->firstname = 'Jan';
- **Severity:** medium
- **Evidence:** *Field report*, seen in 3 plugins.
