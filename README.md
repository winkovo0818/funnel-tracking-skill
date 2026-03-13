# funnel-tracking

Project-local OpenCode skill package for building reusable funnel-style tracking.

## Quick Start

To use this skill, provide:

1. **Webhook URL** (e.g., Feishu Bitable webhook)
2. **Business flow** (e.g., visit → login → submit)
3. **Project type** (Web/Mini-program/App)

AI will generate:
- Complete tracker module
- Event tracking calls at key points
- Environment configuration
- Test guidelines

See "How to Use This SKILL" section in SKILL.md for detailed instructions.

## Included Files

- `SKILL.md`: the actual skill instructions loaded by the agent
- `README.md`: package overview and installation notes
- `EXAMPLES.md`: example prompts and expected outcomes
- `LICENSE.txt`: local license reference used by the skill frontmatter

## What This Skill Covers

- Unified tracking payload design across frontend and backend
- `anonymous_id` / `session_id` / `flow_id` context modeling
- `trackEvent` vs `trackEventOnce` split
- Best-effort client delivery with optional retry queue
- Central backend `/api/track` ingestion or direct Webhook integration
- Single-table approach (event_logs) for simplicity
- Feishu Bitable Webhook integration (recommended for small-medium scale)
- Cross-platform field mapping (Web, H5, Mini-program, App)
- Privacy compliance checklist and data retention policies
- Comprehensive test cases for production readiness

## Recommended Storage Solutions

### Feishu Bitable (Recommended for < 10k events/day)
- Single-table approach (event_logs only)
- Webhook integration (no token management)
- Visual interface for non-technical users
- Built-in views and filters

### Professional Databases (For > 10k events/day)
- Dual-table approach (event_logs + flow_summary)
- PostgreSQL, MySQL, ClickHouse, Doris
- Complex SQL analysis capabilities

## Recommended Directory Shape

```text
.opencode/
  skills/
    funnel-tracking/
      SKILL.md
      README.md
      EXAMPLES.md
      LICENSE.txt
```

## How To Use

If your OpenCode setup supports project-local skills, keep this folder as-is.

If your setup only scans the global skill directory, copy this folder to:

```text
C:\Users\GOC\.config\opencode\skills\funnel-tracking
```

Then load it by name as:

```text
funnel-tracking
```

## Best Invocation Scenarios

Use this skill when the task is about:

- adding funnel tracking to a product flow
- integrating with Feishu Bitable via Webhook
- aligning Web, App, H5, or mini-program tracking contracts
- building a generic tracker helper with retry queue
- refactoring ad hoc business events into a reusable schema
- designing single-table event storage with views

## Expected Outputs

When used well, the skill should help produce:

- a clear event enum and field contract
- a client tracker module with retry queue support
- tracking calls at key business points (login, submit, etc.)
- environment configuration (.env or config file)
- a Feishu Bitable table schema and view recommendations
- cross-platform field mapping guidelines
- privacy compliance checklist
- comprehensive test cases
- rollout, validation, and debugging notes

## Example Usage

```text
I want to add tracking to my project using Feishu Bitable.

Webhook URL: https://your-webhook-url.com
Token: xxxxxxx

Business flow:
1. Visit homepage
2. Login
3. Start assessment
4. Submit result

Project type: Web (React)
```

AI will automatically:
- Create `src/utils/tracker.js`
- Add tracking calls in login/submit pages
- Generate `.env` configuration
- Provide Feishu view setup instructions
