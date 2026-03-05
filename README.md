# LinkedIn Automation Skills for Claude

Two Claude skills for LinkedIn Sales Navigator: one builds targeted lead lists, the other sends personalized InMail outreach.

## Skills

### `sales-nav-list-builder`

Builds lead lists through a guided conversation. You describe who you're looking for (titles, industries, geography, company size, seniority) and the skill applies the filters in Sales Navigator and saves the results as a named list — page by page.

Supports multi-page saving (up to 1,500 leads), search saving, list deletion, and excluding leads already in other lists or previously contacted.

### `linkedin-outreach`

Sends personalized InMail messages to leads from a Sales Navigator list. Reads your lead list, filters to contacts with no prior activity, and sends a templated message with per-contact placeholder replacement.

Supports Open Profile detection — can message only free-to-contact leads, or message everyone and automatically switch to Open Profile–only mode when InMail credits run out (instead of stopping).

## Prerequisites

- [Claude Code](https://claude.com/claude-code) OR [Claude Desktop](https://claude.com/download)
- [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/) extension (connected)
- LinkedIn Sales Navigator subscription

## Quick Start

### Option A: Claude Code (CLI)

Install both skills as personal skills:
```bash
mkdir -p ~/.claude/skills/sales-nav-list-builder
cp sales-nav-list-builder/SKILL.md ~/.claude/skills/sales-nav-list-builder/SKILL.md

mkdir -p ~/.claude/skills/linkedin-outreach
cp linkedin-outreach/SKILL.md ~/.claude/skills/linkedin-outreach/SKILL.md
```

Then start Claude Code and run:
```
/sales-nav-list-builder
```
or
```
/linkedin-outreach
```

### Option B: Claude Desktop (GUI)

1. Open the Claude desktop app, go to Settings -> Capabilities -> scroll down to Skills -> Add
2. Upload the `SKILL.md` from the skill folder you want to use
3. Start a Cowork session and mention the skill by name

## Message Template (Outreach)

Your outreach message template is saved to `core_message.txt` and sent **verbatim** to each contact. Use bracket placeholders for per-contact data:

| Placeholder | Type | Replaced With |
|---|---|---|
| `[First Name]` | Data | Contact's first name |
| `[Company]` | Data | Current company |
| `[Any Custom]` | Data | Resolved from profile data |
| `[Write a short ice-breaker about their role]` | Instruction | Claude generates content based on the guidance inside the brackets |

## How It Works

**List Builder:**
1. Guided interview to collect targeting criteria
2. Applies filters in Sales Navigator
3. Saves results page by page to a named lead list

**Outreach:**
1. Navigates to the lead list, fetches all profile URLs, filters to "No activity" contacts
2. Opens each profile, reads details, composes InMail with placeholder replacement, sends, verifies delivery
3. Detects Open Profile vs credit-required per lead — skips or proceeds based on your preference
4. Reports a summary of sent, skipped, and failed contacts

## Safety

- Confirms with the user before starting and at session limits
- Verifies recipient name matches before every send
- Stops immediately on rate limits, captchas, or monthly send limits
- Monitors InMail credits — switches to Open Profile–only mode when exhausted instead of stopping
- Skips contacts who have disabled communications
