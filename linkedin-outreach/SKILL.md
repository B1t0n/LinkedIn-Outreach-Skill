---
name: linkedin-outreach
description: Send personalized LinkedIn InMail messages to leads from a Sales Navigator list. Walks the user through setup (browser check, leads URL, core message draft), then executes the outreach workflow. Use when asked to outreach, message, or contact LinkedIn leads.
license: MIT
---

# LinkedIn Outreach Skill

Send personalized InMail messages to leads from a LinkedIn Sales Navigator list.

---

## Phase 0: Interactive Setup

Before executing the outreach workflow, walk the user through each dependency. Do NOT skip ahead — confirm each step before moving on.

### Step 0.1 — Verify Browser Connection

Check that the Claude in Chrome extension is connected:
- Call `tabs_context_mcp` to verify
- If no connection, tell the user: "The Claude in Chrome extension must be connected. Please ensure it's installed and click Connect in Chrome."
- Do not proceed until connected.

### Step 0.2 — Confirm LinkedIn Login

Ask the user: "Are you logged into LinkedIn Sales Navigator in the connected Chrome browser?"
- If unsure, offer to navigate to linkedin.com to check.
- Do not proceed until confirmed.

### Step 0.3 — Collect Leads List URL

If `$ARGUMENTS` was provided, use it as the leads list URL.

If no URL was provided, ask the user:
"Please paste your LinkedIn Sales Navigator lead list URL."
- Wait for the user to provide the URL.
- Validate it looks like a LinkedIn Sales Navigator URL.

### Step 0.4 — Collect or Load Core Message

Check if `core_message.txt` exists in the project root.

**If it exists:** Read it, show the user a preview, and ask: "This is your current message template. Use this, or would you like to draft a new one?"

**If it does NOT exist:** Ask the user:
"Please draft your core outreach message. You can use placeholders in `[Brackets]` that will be replaced per contact from their profile data. Common placeholders:
- `[First Name]` — contact's first name
- `[Company]` — current company
- `[Title]` — job title
- Any other `[Custom Placeholder]` — will be resolved from profile details

The message is sent VERBATIM except for placeholder replacement.

Paste your message below:"

Once the user provides the message:
1. Parse the message for all `[Placeholder]` tokens and list them back to the user
2. Save it to `core_message.txt` in the project root
3. Show the saved message back to the user for confirmation
4. Ask: "Message saved with X placeholders: [list them]. Does this look correct?"
5. Only proceed after confirmation.

### Step 0.4b — Collect Subject Line

Ask the user:
"What subject line should be used for InMail messages?"
- Save their response for use in Step 2.6.
- If the user wants different subjects per contact, note that, but default is a single subject for all.

### Step 0.4c — Open Profile Preference

Ask the user using AskUserQuestion: **"Do you want to message only leads with Open Profile (free InMail), or all leads?"**

Options:
- **Open Profile only** — Only message leads where the compose window shows "Free to Open Profile". Skip any lead that would cost an InMail credit. This preserves credits entirely.
- **All leads** — Message everyone regardless of Open Profile status, using InMail credits as needed. If credits run out mid-session, automatically switch to Open Profile–only mode for remaining contacts (skipping credit-required leads instead of stopping).

Store this preference for use in Step 2.3b.

### Step 0.4d — Session Size

Ask the user: **"How many leads do you want to message in this session?"**

The user can provide any number (e.g., "all of them", "50", "just 10 to test"). Accept their answer as-is. If they say "all", use the full list. Default cap is 100 per session to avoid rate limits — if the user requests more, warn them about LinkedIn's daily/monthly send limits but respect their choice.

### Step 0.5 — Confirm and Begin

Summarize the setup:
```
Ready to begin outreach:
- Browser: Connected
- LinkedIn: Logged in
- Lead list: [URL]
- Message template: Loaded (X words, Y placeholders)
- Subject line: [subject]
- Open Profile: [only / all leads (auto-switch when credits exhausted)]
- Session size: [number or "all"] (recommended cap: 100)
```

Ask: "Ready to start? I'll navigate to the lead list and begin processing contacts with 'No activity'."

Wait for explicit confirmation before proceeding to Phase 1.

---

## Phase 1: Pre-Flight (Once Per Session)

### Step 1.1 — Get Browser Tab
```
Use tabs_context_mcp to get available tabs.
If no tabs exist, use tabs_create_mcp to create one.
```

### Step 1.2 — Load Message Template
```
Read core_message.txt from the project root.
Store the full message text for use throughout the session.
The message body is IMMUTABLE — never modify it.
Only [Placeholder] tokens in brackets are replaced with profile data.
```

### Step 1.3 — Navigate to Lead List
```
Use navigate tool to go to the lead list URL.
Wait for the page to fully load.
```

### Step 1.4 — Batch-Fetch All Profile URLs

Use `javascript_tool` to extract all profile URLs from the list in a single call:

```javascript
const links = document.querySelectorAll('a[href*="/lead/"]');
const profiles = Array.from(links).map(a => ({
  name: a.textContent.trim(),
  url: a.href
})).filter(p => p.name && p.url);
JSON.stringify(profiles, null, 2);
```

### Step 1.5 — Check Outreach Activity for All Contacts

Read the "Outreach activity" column for every contact. ONLY message contacts showing "No activity". Skip any with "Message sent" or other activity.

Use `javascript_tool`:
```javascript
const rows = document.querySelectorAll('table tbody tr');
const contacts = Array.from(rows).map(row => {
  const nameEl = row.querySelector('a[href*="/lead/"]');
  const cells = Array.from(row.querySelectorAll('td'));
  let activity = 'Unknown';
  for (const cell of cells) {
    const text = cell.textContent.trim();
    if (text.includes('No activity')) { activity = 'No activity'; break; }
    if (text.includes('Message sent')) { activity = 'Message sent'; break; }
  }
  return {
    name: nameEl?.textContent?.trim() || 'Unknown',
    url: nameEl?.href || '',
    activity
  };
});
JSON.stringify(contacts, null, 2);
```

Filter to only contacts where activity is "No activity".

### Step 1.6 — Build Contact Queue

Create an ordered list of contacts to message. Track:
- Contact name
- Profile URL
- Status (pending / sent / skipped_activity / skipped_not_open_profile / skipped_no_credits / skipped_comms_disabled / failed)

Report to the user: "Found X contacts with no activity out of Y total. Ready to begin?"
Wait for user confirmation before proceeding.

---

## Phase 2: Per-Contact Messaging Loop

For each contact in the queue:

### Step 2.1 — Navigate to Profile
```
Use navigate tool to go directly to the contact's profile URL.
Do NOT click through the list panel — navigate directly.
Wait for profile to load.
```

### Step 2.2 — Read Profile Details

Use `javascript_tool` to extract profile info:

```javascript
// Primary: parse page title ("First Last | Sales Navigator")
const pageTitle = document.title;
const name = pageTitle.split('|')[0].trim();
const firstName = name.split(' ')[0];

// Fallback selectors for title/company/about (inspect live DOM if these break)
const title = document.querySelector('.profile-topcard__summary-position')?.textContent?.trim() ||
              document.querySelector('[data-anonymize="title"]')?.textContent?.trim() || '';
const company = document.querySelector('.profile-topcard__summary-position a')?.textContent?.trim() ||
                document.querySelector('[data-anonymize="company-name"]')?.textContent?.trim() || '';
const about = document.querySelector('.profile-topcard__summary-content')?.textContent?.trim() || '';

JSON.stringify({ name, firstName, title, company, about }, null, 2);
```

If selectors return empty, fall back to scanning `document.body.innerText` for role/company patterns.
Extract the contact's first name from the full name.

### Step 2.3 — Click Message Button
```
Use find tool with query "Message" to locate the Message button.
Use computer tool with left_click on the button coordinates.
```

### Step 2.3b — Detect Open Profile vs Credit Required

After clicking Message, check whether this lead has Open Profile (free) or requires an InMail credit:

```javascript
const allText = document.body.innerText;
const creditMatch = allText.match(/Use (\d+) of (\d+) credits?/i);
const isOpenProfile = /Free.{0,15}Open\s*Profile/i.test(allText);
JSON.stringify({
  isOpenProfile,
  requiresCredit: !!creditMatch,
  creditsRemaining: creditMatch ? parseInt(creditMatch[2]) : null,
  detail: isOpenProfile ? 'Free to Open Profile' : (creditMatch ? creditMatch[0] : 'unknown')
});
```

- **"Free to Open Profile"** → Lead has Open Profile, free to message. Always proceed.
- **"Use 1 of X credits"** → Lead requires an InMail credit.

**Decision logic based on user preference (Step 0.4c):**

1. **If user chose "Open Profile only":**
   - Open Profile → proceed to Step 2.4
   - Credit required → close compose, skip this contact (status: `skipped_not_open_profile`), move to next

2. **If user chose "All leads":**
   - Open Profile → proceed to Step 2.4
   - Credit required AND credits > 0 → proceed to Step 2.4
   - Credit required AND credits = 0 → close compose, skip this contact (status: `skipped_no_credits`), move to next. Log: "Switched to Open Profile–only mode (credits exhausted)." Continue processing remaining contacts — do NOT stop the session.

### Step 2.4 — Wait for Compose Form

First, check if the contact has disabled communications:
```javascript
const pageText = document.body.innerText;
if (pageText.includes('disabled all communications')) {
  JSON.stringify({ status: 'skip', reason: 'comms_disabled' });
}
```
If comms are disabled, skip this contact immediately — no retries needed.

Otherwise, verify the compose form loaded:
```
Use read_page or javascript_tool to verify:
- Subject input field exists
- Message textarea exists
If not present after a few seconds, wait and retry (up to 3 times).
If still empty after retries, STOP — likely soft rate limit.
```

### Step 2.5 — Verify Recipient (CRITICAL)

**Before composing, verify the conversation header matches the contact name.**

Use `javascript_tool`:
```javascript
const header = document.querySelector('.conversation-insights__header-name')?.textContent?.trim() ||
               document.querySelector('[data-anonymize="person-name"]')?.textContent?.trim() || '';
header;
```

If the header name does NOT match the expected contact name:
1. Close the conversation
2. Retry opening the message
3. If it still doesn't match, skip this contact and report the mismatch

### Step 2.6 — Set Subject Line
```
Use form_input on the subject field.
Value: Use the subject line collected during Step 0.4b.
```

### Step 2.7 — Compose Message Body

**Re-read core_message.txt BEFORE composing** (ensures you always use the latest version).

Replace all `[Placeholder]` tokens with data from the contact's profile:
- `[First Name]` → contact's first name (extracted from full name)
- `[Company]` → current company
- `[Title]` → job title
- Any other `[Custom Placeholder]` → resolve from profile details (name, title, company, about section)
- If a placeholder cannot be resolved from profile data, leave it blank or skip — do NOT guess

Personalization rules:
- The entire core_message.txt body is sent VERBATIM — only `[Placeholder]` tokens in brackets are replaced with profile data
- Optional personalization is ONLY allowed as a brief line between the greeting and the message body

Use the **native input setter pattern** to set the message body (required because React/LinkedIn intercepts normal value assignment):

```javascript
const msg = `YOUR_COMPOSED_MESSAGE_HERE`;

const msgBody = document.querySelector('textarea[aria-label*="message"]') ||
                document.querySelector('textarea[aria-label*="Message"]') ||
                document.querySelectorAll('textarea')[1];

const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
  window.HTMLTextAreaElement.prototype, 'value'
).set;

nativeInputValueSetter.call(msgBody, msg);
msgBody.dispatchEvent(new Event('input', { bubbles: true }));
msgBody.dispatchEvent(new Event('change', { bubbles: true }));
```

### Step 2.8 — Send Message
```
Use find tool to locate the "Send" button.
Use computer tool with left_click to send.
```

### Step 2.9 — Verify Send Success

Use `javascript_tool` to check:

```javascript
const pageText = document.body.innerText;
const hasSendFailed = pageText.includes('Send failed');
const hasLimitReached = pageText.includes('reached the limit');
const hasAwaitingReply = pageText.includes('Awaiting reply');
JSON.stringify({
  sendFailed: hasSendFailed,
  limitReached: hasLimitReached,
  awaitingReply: hasAwaitingReply,
  success: hasAwaitingReply && !hasSendFailed
}, null, 2);
```

**Verification logic:**
- Success = "Awaiting reply" present AND "Send failed" absent
- If "Send failed" or "reached the limit": close conversation, STOP all sending, report to user
- "Awaiting reply" alone is NOT enough (can linger from a previous conversation)

### Step 2.10 — Check InMail Credits

```javascript
const creditText = document.body.innerText.match(/Use \d+ of (\d+) credits?/);
const credits = creditText ? parseInt(creditText[1]) : null;
const isFreeOpenProfile = /Free.{0,15}Open\s*Profile/i.test(document.body.innerText);
JSON.stringify({ credits, isFreeOpenProfile }, null, 2);
```

- "Free to Open Profile" contacts don't consume credits — these can always be messaged
- **If credits drop to 0:** Do NOT stop the session. Instead, automatically switch to Open Profile–only mode. Log the switch and continue processing remaining contacts, skipping any that require credits. Report how many were skipped at the end.
- Only stop if BOTH credits are exhausted AND there are no remaining Open Profile leads to attempt

### Step 2.11 — Close and Continue
```
Close the conversation panel.
Navigate directly to the NEXT contact's profile URL.
Do NOT navigate back to the list between contacts.
```

---

## Phase 3: Post-Session

### Step 3.1 — Report Summary

After completing all contacts, report:

```
Outreach Session Summary
========================
Total contacts processed: X
Messages sent: X
  - Open Profile (free): X
  - InMail credit used: X
Skipped (already contacted): X
Skipped (activity detected): X
Skipped (not Open Profile): X
Skipped (no credits remaining): X
Failed: X (list names and reasons)
InMail credits remaining: X

Contacts requiring retry:
- [Name] — [Reason]
```

### Step 3.2 — Session Limit

When the number of contacts processed reaches the session size from Step 0.4d, pause and ask:
"Reached [session size]-contact session limit. Continue with remaining Y contacts?"

Wait for explicit user confirmation before proceeding.

---

## Critical Rules

- The core message body from core_message.txt must be sent **100% VERBATIM** — ZERO word changes, only `[Placeholder]` tokens in brackets are replaced with profile data
- **Always** check "Outreach activity" column = "No activity" before messaging
- **Always** verify conversation header name matches the contact before sending
- **Stop immediately** on any LinkedIn warning, captcha, or rate limit
- Respect the **session size** from Step 0.4d (default cap: 100), then pause and ask for user confirmation before continuing
- If compose form loads empty after retries, stop — likely soft rate limit

## Screenshot Policy

**Do NOT take screenshots during normal operation.** All reading, clicking, and verification must be done via `javascript_tool`, `find`, `read_page`, `form_input`, and `computer` tools.

Screenshots are ONLY for **diagnostics** — when something is stuck, broken, or the DOM returns unexpected results:
1. Take a screenshot to visually understand the current state
2. Study the screenshot to identify the correct selectors, layout, or issue
3. Use what you learned to craft a better `javascript_tool` call or adjust the approach
4. Resume operating via code — do NOT use the screenshot to click or interact

Screenshots are for **learning**, not for **operating**.

## InMail Credit Monitoring

- After each send, check for "Send failed" text AND "reached the limit" banner
- Verify BOTH "Awaiting reply" AND absence of "Send failed" (old "Awaiting reply" text can linger)
- "Free to Open Profile" contacts do NOT consume credits — detected via `Free.{0,15}Open\s*Profile` in compose window text
- "Use 1 of X credits" contacts require an InMail credit — detected via `/Use (\d+) of (\d+) credits?/i`
- **When credits reach 0:** Do NOT stop. Switch to Open Profile–only mode automatically. Continue processing remaining contacts, messaging only Open Profile leads and skipping credit-required ones.
- **When send fails due to monthly limit:** Close conversation, STOP all sending, report to user. Monthly SEND LIMIT is SEPARATE from credits — LinkedIn has a monthly cap independent of credit balance.
- Credits reset monthly

## Error Handling

### LinkedIn Warning / Captcha / Rate Limit
- **Stop immediately** — do not attempt to bypass
- Take a diagnostic screenshot to understand the blocker, then report findings to user
- List remaining unprocessed contacts

### Compose Form Won't Load
- Retry up to 3 times with short waits
- If still empty, this is likely a **soft rate limit**
- Stop sending and report to user

### Name Mismatch in Conversation
- Close the conversation
- Retry once
- If still mismatched, skip the contact and note it in the report

### InMail Send Failure
- Check for "Send failed" and limit banners
- If failure is due to **credits exhausted**: switch to Open Profile–only mode, continue session
- If failure is due to **monthly send limit** or other hard limit: stop ALL sending, report to user
- Report credits remaining and contacts not yet messaged

### Monthly Send Limit
- Separate from InMail credits — LinkedIn has a monthly cap
- If sends fail despite having credits, the monthly limit is likely reached
- Note all remaining contacts for retry next month
