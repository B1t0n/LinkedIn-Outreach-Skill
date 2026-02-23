# LinkedIn Outreach Skill for Claude

A Claude skill that automates personalized LinkedIn InMail outreach through Sales Navigator. It reads a lead list, filters contacts by activity status, and sends templated messages with per-contact placeholder replacement.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) OR [Claude Desktop](https://claude.com/download)
- [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/) extension (connected)
- LinkedIn Sales Navigator subscription with InMail credits

## Quick Start

### Option A: Claude Code (CLI)

**Install as a personal skill (available across all projects):**
```bash
mkdir -p ~/.claude/skills/linkedin-outreach
cp SKILL.md ~/.claude/skills/linkedin-outreach/SKILL.md
```

Then start Claude Code and run:
```
/linkedin-outreach
```

### Option B: Claude Desktop (GUI)

1. Open the Claude desktop app, go to settings -> capabilities -> scroll down to skills -> Add
2. Upload the `SKILL.md` content
3. Start Cowork session, mention the use of linkedin-outreach skill and follow the interactive setup prompts

### Initial Setup
Both options will walk you through:
- Verifying browser connection
- Confirming LinkedIn login
- Collecting your lead list URL
- Drafting a message template with `[Placeholder]` tokens
- Drafting your InMail subject

## Message Template

Your message template is saved to `core_message.txt` and sent **verbatim** to each contact. Use bracket placeholders for per-contact data:

| Placeholder | Type | Replaced With |
|---|---|---|
| `[First Name]` | Data | Contact's first name |
| `[Company]` | Data | Current company |
| `[Any Custom]` | Data | Resolved from profile data |
| `[Write a short ice-breaker about their role in cybersecurity]` | Instruction | Claude generates content based on the guidance inside the brackets |

Instruction placeholders let you embed creative direction directly in your template. Claude reads the contact's profile and follows the guidance to generate that section — while keeping the rest of the message verbatim.

## How It Works

1. **Pre-flight** — Navigates to the lead list, fetches all profile URLs, filters to contacts with "No activity" only
2. **Per contact** — Opens profile, reads details, composes InMail with placeholder replacement, sends, verifies delivery
3. **Post-session** — Reports a summary of sent, skipped, and failed contacts

## Safety

- Confirms with the user before starting and at session limits
- Verifies recipient name matches before every send
- Stops immediately on rate limits, captchas, or send failures
- Monitors InMail credit count and monthly send caps
- Skips contacts who have disabled communications
