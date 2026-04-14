---
name: roborewsty
description: Generate structured workflow specifications for RoboRewsty, the Rewst AI workflow builder. Use when the user wants to build, design, or describe a Rewst workflow, automate IT processes in Rewst, or prompt RoboRewsty to create automation (onboarding, offboarding, ticketing, group management, license assignment, Microsoft 365, ConnectWise Manage, Microsoft Graph integrations).
---

# RoboRewsty Workflow Spec

RoboRewsty is the Rewst AI assistant that builds workflows step by step in the Workflow Builder. Give it a structured spec and it will look up the correct integration actions, validate Jinja2 syntax, and guide you through building.

## How to use this skill

1. Ask the user what the workflow should do (goal, trigger, integrations)
2. Gather inputs, steps, and outputs
3. **Look up exact action names** by fetching the relevant integration's actions & endpoints page from the Rewst docs (see Reference Documentation below) — never guess action names
4. Output a complete spec in the format below
5. Paste the spec into RoboRewsty in Rewst

---

## Spec Format

```
## Workflow: [Workflow Name]

### 1. Goal
[One sentence describing what this workflow automates and why.]

### 2. Trigger
- Type: [Form / Webhook / Cron / Manual]
- Description: [What initiates this workflow]

### 3. Inputs (Form fields or webhook payload)
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| field_name | string | yes | Description |

### 4. Integrations Used
- [Integration name] — [purpose in this workflow]

### 5. Steps
1. **step_name**: [Action description]. Integration: [name]. Key fields: `field = {{ CTX.value }}`.
   - Data alias on transition: `alias = {{ RESULT.result.data.field }}`
2. **step_name**: [Next action]. Condition: [when this runs].
   - with-items: `item in {{ CTX.list }}` — [what it does per item]
3. **end**: Noop convergence point.

### 6. Outputs / Results
- Publish `CTX.variable_name` — [what it contains, used by whom]

### 7. Organization Scope
[Single org / Multi-org / MSP-level]

### 8. Constraints / Notes
- [Idempotency rules, retry logic, timing considerations]
- [Any known API quirks or sequencing requirements]
```

---

## Common Patterns

### Idempotency check
Before creating a resource, add a step to check if it already exists:
```
1. **check_existing**: Query [integration] for the resource by unique identifier.
   - Transition to "create" if not found, skip to "end" if found.
```

### With-items (loop over a list)
```
**add_to_groups**: with-items: `group in {{ CTX.security_groups }}`
- Action: Add user to group via Microsoft Graph
- Key field: `group_id = {{ item }}`
```

### Data aliases (passing results between steps)
```
Data alias on transition from step N:
`new_user_id = {{ RESULT.result.data.id }}`
```

### Delay / retry for async operations
```
**wait_for_license**: Add a delay task (e.g. 30s) or a retry loop checking license assignment status before proceeding.
```

### End noop — Task Transition Criteria

Every end noop needs Task Transition Criteria Sensitivity set to 1.
RoboRewsty does NOT pick this up from a spec note — write it as an explicit click instruction in the end step:

```
**end**: Noop convergence point.
  After placing the end noop: click the noop node → open properties panel →
  find "Task Transition Criteria Sensitivity" → set value to 1.
  Only one path reaches end per execution. Leaving the default (all paths)
  causes TaskTransitionCriteriaError on every run.
```

### Per-org webhook trigger (MSP option generators)

When building a workflow that syncs data per org (e.g. users, devices), use a
webhook trigger. The org context is set automatically via the orgId in the URL:
`https://engine.rewst.eu/webhooks/custom/trigger/{triggerId}/{rewstOrgId}`

No parent/child multi-org loop needed — Microsoft Graph and ORG.VARIABLES
resolve to the correct org automatically.

### Filtering lists safely

Prefer explicit list comprehensions over `selectattr` for nullable fields.
`selectattr('field')` silently passes items where the field is missing entirely.

```
# Unreliable — passes items where assignedLicenses is missing
{{ CTX.users | selectattr('assignedLicenses') | list }}

# Reliable — explicitly checks existence and non-empty
{{ [u for u in CTX.users if u.get('assignedLicenses') and u.get('assignedLicenses') | length > 0] }}
```

---

## Example: New User Onboarding

```
## Workflow: New User Onboarding

### 1. Goal
Create a new Microsoft 365 user, assign a license, add to security groups, and open an onboarding ticket in ConnectWise Manage.

### 2. Trigger
- Type: Form
- Description: IT staff submits the new hire form

### 3. Inputs
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| first_name | string | yes | |
| last_name | string | yes | |
| email | string | yes | UPN for M365 |
| license_sku | string | yes | M365 license SKU ID |
| security_groups | list | no | Azure AD group IDs |

### 4. Integrations Used
- Microsoft Graph — create user, assign license, add to groups
- ConnectWise Manage — create onboarding service ticket

### 5. Steps
1. **check_user_exists**: Query Microsoft Graph GET /users/{{ CTX.email }}. 
   - Transition to "create_user" if 404, skip to "end" if user exists.
2. **create_user**: POST /users via Microsoft Graph with displayName, userPrincipalName, password.
   - Data alias on transition: `new_user_id = {{ RESULT.result.data.id }}`
3. **assign_license**: POST /users/{{ CTX.new_user_id }}/assignLicense with `license_sku`.
4. **add_to_groups**: If `security_groups` is not empty, with-items: `group in {{ CTX.security_groups }}` — POST /groups/{{ item }}/members.
5. **create_onboarding_ticket**: Create service ticket in ConnectWise Manage with summary: "Onboarding: {{ CTX.first_name }} {{ CTX.last_name }} ({{ CTX.email }})".
   - Data alias on transition: `ticket_id = {{ RESULT.result.data.id }}`
6. **end**: Noop convergence point.

### 6. Outputs / Results
- Publish `CTX.new_user_id` — M365 user ID for downstream workflows
- Publish `CTX.ticket_id` — ConnectWise ticket ID for follow-up

### 7. Organization Scope
Single org (the org that submits the form)

### 8. Constraints / Notes
- Idempotent: check_user_exists prevents duplicate accounts
- License assignment may take a moment — consider a 30s delay or retry before proceeding
- security_groups is optional; skip add_to_groups if empty
```

---

## JSON Body — Critical Pitfall

**Never pass a full object as a single Jinja expression in the JSON body key-value UI.**

If you write a value like `{{ CTX.payload }}` where `payload` is a dict, Rewst will iterate over the string representation character by character. You'll see keys 0, 1, 2... with values `{`, `"`, `o`, `r`, `g`... instead of your actual JSON.

**Wrong:**
```
JSON body key-value UI:
  (value field) → {{ CTX.payload }}   ← iterates characters, broken
```

**Correct — map each key individually:**
```
JSON body key-value UI:
  displayName        → {{ CTX.first_name }} {{ CTX.last_name }}
  userPrincipalName  → {{ CTX.email }}
  accountEnabled     → true
```

**Correct — for fully dynamic/nested bodies, switch to Raw JSON mode:**
```
Raw JSON field:
{
  "displayName": "{{ CTX.first_name }} {{ CTX.last_name }}",
  "userPrincipalName": "{{ CTX.email }}",
  "passwordProfile": {{ CTX.password_profile | tojson }}
}
```

Always tell RoboRewsty to map JSON body fields individually, not as a single context variable.

**Raw JSON body — block syntax does not work:**

Never use `{% for %}{% if %}{% endfor %}` block tags in a Generic HTTP Raw JSON body.
Rewst evaluates the body as a Jinja2 expression, not a template. Use a list comprehension:

```
# Wrong — TemplateSyntaxError
{
  "items": [
    {% for x in CTX.list %}{"a": "{{ x.a }}"}{% endfor %}
  ]
}

# Correct — list comprehension in dict expression
{{ {"orgId": CTX.organization.id, "items": [{"a": x.a, "b": x.b} for x in CTX.list]} }}
```

**`| tojson` behavior is inconsistent:**

Sometimes the dict expression renders as valid JSON without `| tojson`, sometimes it renders
as Python single-quote syntax (invalid JSON). If the endpoint returns an "invalid JSON" error,
add `| tojson` to the end of the expression:

```
{{ {"orgId": CTX.organization.id, "items": [...]} | tojson }}
```

Endpoints that do a double `JSON.parse` (parse string → parse again if still a string) handle
both cases correctly.

---

## Tips for RoboRewsty

- Always name steps with snake_case
- Use `{{ CTX.variable }}` for context variables, `{{ RESULT.result.data.field }}` for action results
- Use `{{ CTX.organization.id }}` for the current org ID — `ORG.ID` and `ORG.VARIABLES.rewst_org_id` do not exist in Rewst Jinja context
- Specify the integration name exactly as it appears in Rewst (e.g. "Microsoft Graph", "ConnectWise Manage")
- List every data alias explicitly — RoboRewsty uses these to wire transitions
- Note conditions on transitions (e.g. "only if list is not empty")
- In JSON body fields: always map keys individually, never pass a whole object as one expression

---

## Reference Documentation

When building a spec, **always look up** the official Rewst docs to get exact action names, required fields, and integration-specific details. Do not guess action names — they are documented.

> **Fetching tip**: `docs.rewst.help` blocks direct web fetches (403). Use the raw GitHub source instead:
> `https://raw.githubusercontent.com/RewstApp/docs.rewst.help/main/<path>.md`
> For example, to fetch the Microsoft Graph actions page:
> `https://raw.githubusercontent.com/RewstApp/docs.rewst.help/main/documentation/integrations/integration-guides/microsoft-cloud-integration-bundle/microsoft-cloud-integration-bundle-actions-and-endpoints.md`

### Rewst Documentation (docs.rewst.help)

| Topic | URL | Raw GitHub path |
|-------|-----|-----------------|
| Workflow best practices | https://docs.rewst.help/documentation/automations/workflows/best-practices-for-designing-workflows | `documentation/automations/workflows/best-practices-for-designing-workflows.md` |
| Workflow builder setup | https://docs.rewst.help/documentation/automations/workflows/workflow-builder-how-to-set-up-a-workflow | `documentation/automations/workflows/workflow-builder-how-to-set-up-a-workflow.md` |
| Data aliases | https://docs.rewst.help/documentation/automations/workflows/data-aliases | `documentation/automations/workflows/data-aliases.md` |
| Input & context variables | https://docs.rewst.help/documentation/automations/workflows/data-input-and-output-input-variables-and-context-variables | `documentation/automations/workflows/data-input-and-output-input-variables-and-context-variables.md` |
| Completion handlers & wrappers | https://docs.rewst.help/documentation/automations/workflows/completion-handlers-and-workflow-wrappers | `documentation/automations/workflows/completion-handlers-and-workflow-wrappers.md` |
| Jinja essentials | https://docs.rewst.help/documentation/jinja/jinja-essentials | `documentation/jinja/jinja-essentials.md` |
| Jinja filters (full list) | https://docs.rewst.help/documentation/jinja/list-of-jinja-filters | `documentation/jinja/list-of-jinja-filters.md` |
| Internal Jinja examples | https://docs.rewst.help/documentation/jinja/internal-rewst-jinja-examples | `documentation/jinja/internal-rewst-jinja-examples.md` |
| Rewst actions | https://docs.rewst.help/documentation/automations/actions-in-rewst/rewst-actions | `documentation/automations/actions-in-rewst/rewst-actions.md` |
| Integration guides | https://docs.rewst.help/documentation/configuration/integrations | `documentation/configuration/integrations/` |
| Custom integrations | https://docs.rewst.help/documentation/configuration/integrations/custom-integrations | `documentation/configuration/integrations/custom-integrations.md` |

### Integration Actions & Endpoints

Look these up to get **exact action names** (e.g. "List Users", "Create User") for a specific integration.

| Integration | Actions & endpoints URL | Raw GitHub path |
|-------------|------------------------|-----------------|
| Microsoft Graph | https://docs.rewst.help/documentation/integrations/integration-guides/microsoft-cloud-integration-bundle/microsoft-cloud-integration-bundle-actions-and-endpoints | `documentation/integrations/integration-guides/microsoft-cloud-integration-bundle/microsoft-cloud-integration-bundle-actions-and-endpoints.md` |
| ConnectWise PSA | https://docs.rewst.help/documentation/integrations/psa/connectwise-manage/actions-and-endpoints | `documentation/integrations/psa/connectwise-manage/actions-and-endpoints.md` |
| Core actions | https://docs.rewst.help/documentation/actions-in-rewst/core-actions | `documentation/actions-in-rewst/core-actions.md` |

For other integrations, search the docs repo or use web search: `site:docs.rewst.help <integration name> actions endpoints`

### Cluck University (learn.rewst.io)

| Topic | URL |
|-------|-----|
| Course catalog | https://learn.rewst.io/page/course-catalog |
| Rewst Foundations path | https://learn.rewst.io/path/rewst-foundations |
| Automation Basics path | https://learn.rewst.io/path/automation-basics-path |
| Building a basic form and workflow | https://learn.rewst.io/building-a-basic-form-and-workflow |
| Live training sessions | https://learn.rewst.io/page/live-training |

### When to look things up

- **Integration actions**: fetch the actions & endpoints page for that integration to get exact action names and parameters — do not guess
- **Unfamiliar Jinja filter**: fetch the full filter list before inventing syntax
- **Form or trigger setup**: consult the workflow builder setup page
- **Subworkflow patterns**: check completion handlers & wrappers documentation
- **Training walkthroughs**: point users to the relevant Cluck University course
- **Permissions or setup issues**: check the integration setup guide for prerequisites (e.g. Azure AD Premium for signInActivity)

---

## Rewst Variable Roots

These are the top-level variable namespaces available inside Jinja expressions in Rewst workflows.

| Root | Purpose | Example |
|------|---------|---------|
| `CTX` | Context — workflow inputs, data aliases, task results | `{{ CTX.user_id }}` |
| `ORG` | Organization — org-level config and metadata | `{{ ORG.VARIABLES.company_name }}` |
| `ORG.ATTRIBUTES` | Organization ID and managing org ID | `{{ ORG.ATTRIBUTES.id }}` |
| `ORG.MAPPING` | External system identifiers for the org | `{{ ORG.MAPPING.psa_id }}` |
| `ORG.HAS_TAG` | Check org tags (spaces → underscores) | `{{ ORG.HAS_TAG.premium_support }}` |
| `TASKS` | Reference previous task results by name | `{{ TASKS.list_users.result.result }}` |
| `TASKS_RESULT_DATA` | Shortcut for `TASKS.<name>.result.result` | `{{ TASKS_RESULT_DATA }}` |
| `RESULT` | Current task's result (in data alias context) | `{{ RESULT.result.data.id }}` |
| `UTILS` | Utilities — timestamps, UUIDs | `{{ UTILS.uuid4() }}` |
| `WORKFLOW` | Current workflow metadata | `{{ WORKFLOW.id }}` |
| `USER` | Executing user info | `{{ USER.username }}` |

**Important**: for with-items tasks, use `TASKS.<task_name>.collected_results` instead of `.result.result`.

### UTILS functions

- `{{ UTILS.NOW() }}` — current timestamp (supports timezone and format args)
- `{{ UTILS.uuid4() }}` — generate a random UUID
- `{{ now('utc', '%Y-%m-%d') }}` — current date in UTC

---

## Commonly Used Jinja Filters

These are the filters most relevant when building workflow specs. For the full list, see the Jinja filters reference above.

### Text
`lower`, `upper`, `title`, `capitalize`, `trim`, `replace`, `truncate`, `wordcount`

### Type conversion
`int`, `float`, `string`, `list`

### JSON / data format
- `json_dump` / `json` — serialize to JSON string
- `json_parse` / `from_json_string` — deserialize JSON
- `to_yaml_string` / `yaml_parse` — YAML conversion
- `csv` / `parse_csv` — CSV conversion

### Collections
`first`, `last`, `length`, `sort`, `reverse`, `unique`, `flatten`, `join`, `map`, `select`, `reject`, `zip`, `sum`, `min`, `max`

### Defaults & safety
- `d` / `default` — fallback value if undefined: `{{ CTX.name | d("Unknown") }}`
- `| d` is the idiomatic Rewst way to safely reference optional variables

### Date & time (Rewst-specific)
- `as_datetime` — parse string to datetime
- `as_timezone("US/Eastern")` — convert timezone
- `format_datetime("%Y-%m-%d")` — format output
- `datedelta(days=7)` — add/subtract time
- `convert_from_epoch` — epoch to datetime

### Encoding
`base64`, `decode_base64`, `urlencode`, `urldecode`, `escape`

### Regex
`regex_match`, `regex_search`, `regex_findall`, `regex_replace`

### Rewst extras
- `combine` — merge dicts: `{{ dict_a | combine(dict_b) }}`
- `is_type("int")` — check value type
- `is_json` — validate JSON
- `hmac("secret")` — HMAC digest
- `version_bump_major` / `_minor` / `_patch` — semver increment

---

## Workflow Design Best Practices

These practices come from the official Rewst documentation and field experience.

### Naming conventions
- Workflow names: clear and descriptive (e.g. "List Disabled User Accounts"), avoid personal identifiers
- Variables: pick one convention (snake_case or camelCase) and stick to it across all workflows
- Prefix integration-specific variables: `psa_ticket_id`, `graph_user_id`
- Use tags for additional workflow organization

### Subworkflows
- Use a subworkflow with With Items as a best practice when iterating through multiple items
- This makes each iteration independently debuggable and keeps the parent workflow clean

### API response optimization
- First run a query without field filtering to discover the full response structure
- Then limit the response to only the fields you need — keeps workflows clean and reduces data overhead

### Task transitions
- Use conditional logic: `{{ SUCCEEDED and CTX.list_of_things | d }}`
- Conditions evaluate left to right
- "Follow All" when multiple paths may execute simultaneously
- "Follow First" when only one condition should trigger

### Data alias hygiene
- Separate complex alias creation into dedicated "Set Variable" noop tasks rather than embedding in API action transitions
- This simplifies troubleshooting and lets you test with actual data

### Testing & documentation
- Develop and test in a sandbox or development environment first
- Always test a workflow before publishing
- Document workflow changes during updates for version control
- Use RoboRewsty or manual documentation to track the intent of each action

### Try-catch for error handling
Rewst supports try-catch in Jinja for graceful error handling:
```
{{ try }}
    {{ CTX.possibly_undefined_variable }}
{{ catch }}
    Error: {{ exception }}
{{ endtry }}
```

---

## Changelog

- 2026-04-14: Add raw GitHub fetch paths and integration actions & endpoints lookup table for accurate action names
- 2026-04-14: Add Rewst docs & Cluck University references, variable roots, Jinja filters, and expanded best practices
- 2026-04-14: Initial version — spec format, common patterns, JSON body pitfall, new user onboarding example
