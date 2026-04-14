---
name: roborewsty
description: Generate structured workflow specifications for RoboRewsty, the Rewst AI workflow builder. Use when the user wants to build, design, or describe a Rewst workflow, automate IT processes in Rewst, or prompt RoboRewsty to create automation (onboarding, offboarding, ticketing, group management, license assignment, Microsoft 365, ConnectWise Manage, Microsoft Graph integrations).
---

# RoboRewsty Workflow Spec

RoboRewsty is the Rewst AI assistant that builds workflows step by step in the Workflow Builder. Give it a structured spec and it will look up the correct integration actions, validate Jinja2 syntax, and guide you through building.

## How to use this skill

1. Ask the user what the workflow should do (goal, trigger, integrations)
2. Gather inputs, steps, and outputs
3. Output a complete spec in the format below
4. Paste the spec into RoboRewsty in Rewst

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

---

## Tips for RoboRewsty

- Always name steps with snake_case
- Use `{{ CTX.variable }}` for context variables, `{{ RESULT.result.data.field }}` for action results
- Specify the integration name exactly as it appears in Rewst (e.g. "Microsoft Graph", "ConnectWise Manage")
- List every data alias explicitly — RoboRewsty uses these to wire transitions
- Note conditions on transitions (e.g. "only if list is not empty")
- In JSON body fields: always map keys individually, never pass a whole object as one expression
