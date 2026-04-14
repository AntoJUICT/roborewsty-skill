# RoboRewsty Skill for Claude Code

A Claude Code skill that generates structured workflow specs for [RoboRewsty](https://rewst.io), the AI workflow builder inside Rewst.

Describe what you want to automate, and Claude will output a complete spec you can paste directly into RoboRewsty.

## Install

### One command (macOS / Linux)

```bash
git clone https://github.com/AntoJUICT/roborewsty-skill ~/.claude/skills/roborewsty
```

### One command (Windows — PowerShell)

```powershell
git clone https://github.com/AntoJUICT/roborewsty-skill "$env:USERPROFILE\.claude\skills\roborewsty"
```

After cloning, restart Claude Code. The skill is active immediately.

## Usage

Once installed, Claude picks up the skill automatically when you ask to build a Rewst workflow. You can also trigger it explicitly with:

```
/roborewsty
```

Or just describe what you want:

> "Build a Rewst workflow that creates an M365 user and opens a ConnectWise ticket"

Claude asks a few questions, then outputs a complete spec ready to paste into RoboRewsty.

## What it does

- Gathers your workflow goal, trigger type, and integrations
- Outputs a structured spec with steps, Jinja2 variables, data aliases, and transition conditions
- Includes common patterns: idempotency checks, with-items loops, delay/retry logic
- Flags known Rewst pitfalls (like the JSON body iteration bug)

Works with any Rewst integration: Microsoft Graph, ConnectWise Manage, Autotask, Halo PSA, and more.

## Example output

```
## Workflow: New User Onboarding

### 1. Goal
Create an M365 user, assign a license, and open an onboarding ticket in ConnectWise Manage.

### 2. Trigger
- Type: Form
- Description: IT staff submits the new hire form

### 5. Steps
1. check_user_exists: GET /users/{{ CTX.email }} — skip if exists
2. create_user: POST /users with displayName, UPN, password
3. assign_license: POST /users/{{ CTX.new_user_id }}/assignLicense
4. create_ticket: Create service ticket in ConnectWise Manage
5. end: Noop convergence point.
```

## Update

```bash
cd ~/.claude/skills/roborewsty && git pull
```

On Windows (PowerShell):

```powershell
cd "$env:USERPROFILE\.claude\skills\roborewsty"; git pull
```

Restart Claude Code after updating.

## Requirements

- [Claude Code](https://claude.ai/code) installed
- A Rewst account with RoboRewsty access

## License

MIT
