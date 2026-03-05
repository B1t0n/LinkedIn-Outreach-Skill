---
name: sales-nav-list-builder
description: >
  Build LinkedIn Sales Navigator lead lists through a guided conversation. Asks the user what kind of leads they're looking for (job titles, industries, company size, geography, seniority, etc.), applies the relevant filters in Sales Navigator, and saves the results as a named lead list. Supports multi-page saving, search saving, and list deletion. Use this skill whenever the user mentions building a lead list, finding prospects, Sales Navigator search, creating a prospect list, lead generation, or wants to find and save leads on LinkedIn — even if they don't explicitly say "Sales Navigator."
---

# LinkedIn Sales Navigator Lead List Builder

Build targeted lead lists in LinkedIn Sales Navigator through a guided, conversational workflow. The skill asks the user what they're looking for, translates that into Sales Navigator filters, and saves the results as a named list — all with minimal screenshots by using DOM queries and JavaScript interaction.

---

## Phase 0: Setup & Browser Check

### Step 0.1 — Verify Browser Connection

Call `tabs_context_mcp` to check for an active browser connection.
- If no connection exists, tell the user: "I need the Claude in Chrome extension to be connected. Please make sure it's installed and click Connect in Chrome."
- Do not proceed until connected.

### Step 0.2 — Navigate to Sales Navigator Search

Navigate to the Sales Navigator lead search page. The URL pattern is:

```
https://www.linkedin.com/sales/search/people
```

If the page redirects to a login or subscription page, ask the user to log in to Sales Navigator manually and let you know when they're ready.

To show all available filters, append `?viewAllFilters=true` to the URL or click the "See all filters" button at the bottom of the filter panel:

```javascript
const seeAllBtn = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent?.trim() === 'See all filters');
if (seeAllBtn) seeAllBtn.click();
```

---

## Phase 1: Guided User Interview

Ask the user what kind of leads they want to find. Use the AskUserQuestion tool to gather their criteria conversationally.

**MANDATORY QUESTIONS:** Steps 1.1 through 1.7 are ALL mandatory. You MUST ask every one of them — do not skip any, even if the user's initial description seems complete. Each question collects a distinct filter that must be explicitly confirmed or declined by the user before proceeding.

### 1.1 — Job titles & seniority

Ask the user: **"What job titles and seniority levels are you targeting?"**

Prompt them for:
- **Job titles** (e.g., CEO, VP of Sales, Marketing Director, CISO)
- **Seniority level** (e.g., C-Suite, VP, Director, Manager)

These are always required — do not proceed without at least one job title.

### 1.1b — Industry

Ask the user: **"What industries should these leads be in?"**

Examples: SaaS, Healthcare, Financial Services, Manufacturing. The user may specify one or multiple. If they say "any" or "doesn't matter", skip the Industry filter.

### 1.1c — Geography

Ask the user: **"What geographic regions or countries should these leads be in?"**

Examples: United States, United Kingdom, Europe, San Francisco Bay Area, DACH region. This is a **mandatory question** — always ask it explicitly. If the user says "any" or "global", skip the Geography filter but confirm that's intentional.

### 1.1d — Company size

Ask the user: **"What company size (by employee headcount) are you targeting?"**

Examples: 1-50, 51-200, 201-1000, 1001-5000, 5000+. The user may specify one or multiple ranges. If they say "any", skip the Company headcount filter.

### 1.2 — Advanced criteria (optional)

Ask if they want to narrow further with:

- **Function** (e.g., Sales, Engineering, Marketing)
- **Current company** or **Past company** (specific companies)
- **Years in current position** or **Years in current company**
- **Years of experience**
- **Company type** (Public, Private, Nonprofit, etc.)
- **Posted on LinkedIn** (recently active leads)
- **Changed jobs** (recently changed roles)
- **School** attended
- **Profile language**
- **Groups** membership

### 1.3 — Existing list membership

Always ask the user this question using AskUserQuestion: **"Should leads that are already saved in your other Sales Navigator lists be included or excluded?"**

Options:
- **Exclude existing** — Use the "Saved leads and accounts" filter set to EXCLUDE to skip leads already saved in other lists, avoiding duplicates.
- **Include all** — Include everyone matching the filters, even if they're in other lists.

To exclude existing leads, use the "Saved leads and accounts" filter in the filter panel. Expand it and select the exclusion option. This ensures the new list only contains fresh leads the user hasn't already captured.

### 1.4 — Prior interaction filtering

Always ask the user this question using AskUserQuestion: **"Should leads you've already interacted with (messaged, viewed, etc.) be included or excluded?"**

Options:
- **Exclude interacted** — Use the "Spotlight" filters or activity-based exclusions to skip leads the user has already messaged or interacted with. This helps ensure the list only contains fresh, uncontacted leads.
- **Include all** — Include everyone matching the filters, regardless of prior interaction.

This is useful for outreach campaigns where the user wants to avoid double-contacting leads.

### 1.5 — Connection & shared experience preferences

Ask the user using AskUserQuestion: **"Do you want to filter by connection degree or shared experiences?"**

Options:
- **No preference** — Include all connection degrees (1st, 2nd, 3rd+).
- **2nd degree only** — Focus on leads with a mutual connection (warmer outreach).
- **3rd+ degree only** — Focus on net-new contacts outside existing network.
- **Past colleagues / Shared experiences** — Use the "Past colleague" and "Shared experiences" toggle filters to find leads with common ground (same past company, school, groups, etc.).

Available connection filters: 1st Degree, 2nd Degree, 3rd Degree+, Group Member, TeamLink
Available toggle filters: Past colleague, Shared experiences

To apply connection degree, expand the "Connection" filter and select the appropriate checkbox options. To apply past colleague or shared experiences, use the toggle switches (Type C filters).

### 1.6 — Target list size

Always ask the user: **"How many leads are you aiming for in this list? (Sales Navigator caps at 1,500)"**

The user can provide any number (e.g., "about 300", "maximum", "50"). Accept their answer as-is — do NOT force them into predefined buckets. Calculate pages needed: `ceil(target / 25)`.

Reference ranges for setting expectations:
- Up to 25 → 1 page, very quick
- 25–100 → 2–4 pages
- 100–500 → 4–20 pages, a few minutes
- 500–1,500 → 20–60 pages, significant time

Each page contains ~25 leads. The skill will save page by page until the target is reached or results are exhausted. Sales Navigator hard cap is 1,500 leads per list.

### 1.7 — List naming

Ask the user: **"What would you like to name this lead list?"**

If they don't have a preference, suggest a descriptive name based on their criteria (e.g., "CEOs - SaaS - US - 50-200 employees").

Also ask for an optional list description.

### 1.8 — Confirm and summarize

Before applying filters, summarize ALL interview answers back to the user. Every line below must be filled in — if a question was answered "any / doesn't matter", show that explicitly:

```
Here's what I'll search for:
- Titles: [titles]
- Seniority: [levels]
- Industries: [industries or "any"]
- Geography: [locations or "global"]
- Company size: [headcount or "any"]
- [any advanced filters from 1.2]
- Existing leads: [exclude / include all]
- Prior interactions: [exclude interacted / include all]
- Connection: [no preference / 2nd degree / 3rd+ / past colleagues]
- Target list size: [exact number] (~[pages] pages)
- List name: "[name]"
```

Wait for user confirmation before proceeding.

---

## Phase 2: Apply Filters

This is the core automation phase. Use JavaScript and DOM interaction to apply each filter — avoid screenshots except for diagnostics.

### How Filters Work in Sales Navigator

Sales Navigator has three types of filter interactions:

**Type A — Typeahead filters** (text input with autocomplete suggestions):
Used for: Current job title, Current company, Past company, Past job title, Geography, Industry, Function, School, Groups, First name, Last name, Company headquarters location, Connections of, Account lists, Lead lists

**Type B — Checkbox/button filters** (click to select from preset options):
Used for: Company headcount, Seniority level, Company type, Years in current company, Years in current position, Years of experience, Profile language, Connection

**Type C — Toggle filters** (on/off switches):
Used for: Changed jobs, Posted on LinkedIn, Past colleague, Shared experiences

### Step 2.1 — Expand a filter

Each filter has an expand button. Click it to reveal the filter's input area:

```javascript
const btn = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent?.trim() === `Expand ${filterName} filter`);
if (btn) btn.click();
```

Wait briefly after clicking for the filter content to render.

### Step 2.2 — Type A: Typeahead filters

For filters like job title, company, geography, industry:

1. **Find the input field** after expanding the filter. The input will be a combobox with a placeholder like "Add current titles", "Add companies", "Add locations", etc.:

```javascript
const input = document.querySelector('input[role="combobox"][placeholder*="Add"]');
```

2. **Type into the field** using the `computer` tool's `type` action (NOT JavaScript value setting — LinkedIn's Ember framework requires real keyboard events to trigger the typeahead).

3. **Wait for suggestions** to appear. Suggestions are `[role="option"]` elements inside an `artdeco-typeahead`:

```javascript
const suggestions = document.querySelectorAll('[role="option"]');
const results = Array.from(suggestions).map(s => ({
  text: s.querySelector('span[aria-hidden="true"]')?.textContent?.trim(),
  element: s
}));
JSON.stringify(results.map(r => r.text));
```

4. **Select the right suggestion** by clicking the "Include" button inside the matching option:

```javascript
const options = document.querySelectorAll('[role="option"]');
for (const opt of options) {
  const text = opt.querySelector('span[aria-hidden="true"]')?.textContent?.trim();
  if (text === 'TARGET_VALUE') {
    const includeBtn = opt.querySelector('[role="button"]');
    if (includeBtn) includeBtn.click();
    break;
  }
}
```

Each suggestion has two `[role="button"]` divs: the first is **Include**, the second is **Exclude**.

5. **Repeat** for multiple values in the same filter (e.g., multiple job titles). The input field remains available after selecting a value — just type the next one.

6. Applied values appear as green pills with an X button for removal.

### Step 2.3 — Type B: Checkbox/button filters

There are TWO sub-types within Type B:

**Type B1 — Simple checkbox buttons** (Company headcount, Company type, Years filters):

After expanding the filter, the options appear as clickable div elements with class `button--fill-click-area`. Each shows a label and a count:

```javascript
// Example: Select "1001-5000", "5001-10,000", and "10,000+" headcount ranges
// NOTE: Headcount text formatting is inconsistent in the DOM — use startsWith matching
const allButtons = document.querySelectorAll('.button--fill-click-area');
const targets = ['1001-5000', '5001-10,000', '10,000+'];
allButtons.forEach(btn => {
  const label = btn.textContent?.trim();
  if (targets.some(t => label.startsWith(t) || label.includes(t))) {
    btn.click();
  }
});
```

Company headcount ranges available: Self-employed, 1-10, 11-50, 51-200, 201-500, 501-1,000, 1,001-5,000, 5,001-10,000, 10,001+

**Note:** The actual DOM text for headcount ranges may differ slightly from the filter labels (e.g., "1001-5000" without commas vs "1,001-5,000" with commas). Use `startsWith` or partial matching to handle inconsistencies.

**Type B2 — Include/Exclude row filters** (Seniority level, Connection):

These filters display each option as a labeled row with separate **Include** and **Exclude** buttons (using `[role="button"]`). They do NOT use `.button--fill-click-area`.

```javascript
// Example: Include "CXO", "VP", and "Director" seniority levels
const targets = ['CXO', 'VP', 'Director'];
// Find all label spans in the filter section
const allSpans = document.querySelectorAll('span');
for (const span of allSpans) {
  const text = span.textContent?.trim();
  if (targets.includes(text)) {
    // Navigate up to the row container, then find the Include button
    let parent = span.parentElement;
    while (parent && !parent.querySelector('[role="button"]')) {
      parent = parent.parentElement;
    }
    if (parent) {
      const buttons = parent.querySelectorAll('[role="button"]');
      // First [role="button"] = Include, Second = Exclude
      if (buttons.length >= 1) buttons[0].click();
    }
  }
}
```

Seniority levels available: Owner, Partner, CXO, VP, Director, Manager, Senior, Entry, Training

**Recommended: Use `find` tool for seniority filters.** The JavaScript span traversal above can be fragile. A more reliable approach is to use the `find` tool with aria-label patterns:

```
Use find tool: query 'Include "CXO" in Seniority level filter'
Use computer tool: left_click on the returned ref
Repeat for each seniority level (e.g., "VP", "Director")
```

The aria-label format is: `Include "[Level] ([count])" in Seniority level filter` — the `find` tool handles partial matching so the exact count doesn't need to be known.

Connection options available: 1st Degree, 2nd Degree, 3rd Degree+, Group Member, TeamLink

### Step 2.4 — Type C: Toggle filters

For filters like "Changed jobs" and "Posted on LinkedIn":

These use toggle switches. Find them by their label text and click:

```javascript
const toggleLabel = Array.from(document.querySelectorAll('label, span'))
  .find(el => el.textContent?.trim()?.startsWith('Changed jobs'));
if (toggleLabel) {
  const toggle = toggleLabel.closest('[class*="filter"]')?.querySelector('input[type="checkbox"], [role="switch"]');
  if (toggle) toggle.click();
}
```

### Step 2.5 — Verify filters applied

After applying all filters, verify the result count appeared:

```javascript
// Check result count — note: Sales Navigator may show "5K+" or "1M+" for large sets
const resultText = document.body.innerText.match(/(\d[\d,]*[KMB]?\+?)\s+results?/i);
const count = resultText ? resultText[1] : 'unknown';
count;
```

Report the result count to the user: "Found [X] leads matching your criteria."

If the count is too high (e.g., 1M+), suggest adding more filters to narrow the search. If too low, suggest relaxing some criteria.

### Step 2.6 — MANDATORY Pre-Save Filter Verification

**CRITICAL: Do NOT proceed to Phase 3 (Save) until this step is complete.**

Before saving any leads, verify that EVERY filter the user agreed to in the interview has been applied. Go through this checklist by reading the active filter pills/states in the DOM:

```javascript
// Read all active filter pills to verify what's applied
const pills = document.querySelectorAll('[class*="filter-pill"], [class*="filter-value"], [class*="applied"]');
const appliedFilters = Array.from(pills).map(p => p.textContent?.trim()).filter(Boolean);
JSON.stringify(appliedFilters);
```

Cross-reference against the interview summary from Step 1.8. Check each item:

```
Pre-Save Filter Checklist:
[ ] Job titles — applied? [list what's active]
[ ] Seniority — applied? [list what's active]
[ ] Industry — applied or intentionally skipped?
[ ] Geography — applied or intentionally skipped?
[ ] Company size — applied or intentionally skipped?
[ ] Existing leads exclusion — applied or user chose "include all"?
[ ] Prior interaction exclusion — applied or user chose "include all"?
[ ] Connection degree — applied or user chose "no preference"?
[ ] Any advanced filters from 1.2 — applied?
```

**If ANY filter that the user requested is NOT applied:**
1. STOP — do not proceed to save
2. Apply the missing filter(s) now
3. Re-run this checklist
4. Only proceed to Phase 3 when all items are confirmed

**Report the checklist to the user** before proceeding: "All filters verified. Ready to start saving leads."

This step exists because it is easy to forget exclusion filters (existing leads, prior interactions) after applying the main targeting filters. Never skip it.

---

## Phase 3: Save to List

### Step 3.1 — Select leads

Click the "Select all" checkbox in the results toolbar to select all visible leads on the current page:

```javascript
// Find and click Select all - it's a checkbox in the toolbar area
const selectAllCB = document.querySelector('input[id*="multi-selector"]');
if (selectAllCB && !selectAllCB.checked) selectAllCB.click();
```

Alternatively use the `find` tool to locate "Select all" checkbox and click it.

Note: "Select all" only selects leads on the current page (typically 25 per page). For larger lists, the skill will need to iterate through pages.

### Step 3.2 — Open "Save to list" dropdown

Click the "Save to list" button in the toolbar. **Important:** Use the `find` tool + `computer` tool's `left_click` action for this — JavaScript `.click()` may not reliably open the dropdown due to LinkedIn's Ember event handling:

```
Use find tool: query "Save to list button"
Use computer tool: left_click on the returned ref
```

Fallback JavaScript approach (less reliable):
```javascript
const saveBtn = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent?.trim() === 'Save to list');
if (saveBtn) saveBtn.click();
```

This opens a dropdown with:
- **RECENTLY USED LIST** section — showing recently used lists
- **YOUR CUSTOM LISTS** section — showing all custom lists
- **"+ Create new list"** button at the bottom

### Step 3.3 — Create new list

Click the "+ Create new list" button:

```javascript
const createBtn = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent?.trim()?.includes('Create new list'));
if (createBtn) createBtn.click();
```

This opens a modal dialog (`artdeco-modal`) with:
- **List name** input: `input.text-input__input` with placeholder "E.g. Q4 Leads"
- **List description** textarea: `textarea.create-list-modal__description`
- **"Create and save"** button (disabled until a name is entered)
- **"Cancel"** button

### Step 3.4 — Fill in list details

Type the list name into the input field. **Important:** Use `find` tool to locate the "List name" input, then use `computer` tool's `triple_click` (to select any existing text) followed by `type` action. The `form_input` tool alone may not trigger Ember's event listeners, leaving the "Create and save" button disabled.

```
1. Use find tool: query "list name input field"
2. Use computer tool: triple_click on the ref (to focus + select all)
3. Use computer tool: type "Your List Name Here"
4. Verify the "Create and save" button is now enabled
```

Selector references for JavaScript fallback:
```javascript
const nameInput = document.querySelector('.artdeco-modal input.text-input__input');
const descInput = document.querySelector('.artdeco-modal textarea.create-list-modal__description');
```

### Step 3.5 — Create and save

Click the "Create and save" button:

```javascript
const createSaveBtn = Array.from(document.querySelectorAll('.artdeco-modal button'))
  .find(b => b.textContent?.trim() === 'Create and save');
if (createSaveBtn && !createSaveBtn.disabled) createSaveBtn.click();
```

### Step 3.6 — Multi-page saving

Sales Navigator shows ~25 leads per page. Use the target count from Step 1.6 to calculate pages needed: `ceil(target / 25)`. For example, a target of 300 leads = 12 pages.

For each additional page after the first:

1. Navigate to the next page:
```javascript
const nextBtn = document.querySelector('button[aria-label="Next"]');
if (nextBtn && !nextBtn.disabled) nextBtn.click();
```
2. Wait for results to load
3. Select all leads on the new page (`input[id*="multi-selector"]`)
4. Click "Save to list" → select the **existing** list name (do NOT create a new list)
5. Repeat until the target count is reached or pages are exhausted

**Important:** Sales Navigator limits list size to 1,500 leads. Report progress to the user after every 5 pages (e.g., "Saved 125 leads so far, continuing...").

---

## Phase 4: Summary & Handoff

### Step 4.1 — Report results

```
Lead List Created Successfully!
================================
List name: [name]
Total leads saved: [count] (target was [target])
Filters applied:
  - Job titles: [titles]
  - Seniority: [levels]
  - Industries: [industries or "any"]
  - Geography: [locations or "global"]
  - Company size: [headcount or "any"]
  - Existing leads: [excluded / included]
  - Prior interactions: [excluded / included]
  - Connection: [filter or "no preference"]
  [any additional filters]

List URL: [link to the list in Sales Navigator]
```

### Step 4.2 — Suggest next steps

Ask the user what they'd like to do next:
- "Would you like to start outreach to this list?" (can hand off to the linkedin-outreach skill)
- "Would you like to refine the filters and create another list?"
- "Would you like to export the list?"

---

## Phase 5: List Management

### Step 5.1 — Save a search

After applying filters, you can save the search for future use (to re-run later with the same criteria):

1. Look for the "Save search" button in the search results toolbar
2. Click it to save the current filter combination
3. Saved searches are accessible from the Sales Navigator homepage under "Saved searches"

### Step 5.2 — Delete a lead list

To delete an existing lead list:

1. Navigate to the Lead Lists page:
```
https://www.linkedin.com/sales/lists/people
```

2. Find the target list and click its **···** (three dots) menu button. **Important:** The `find` tool may not reliably open this dropdown — use coordinate-based clicking with the `computer` tool on the three-dot icon:
```
Use computer tool: screenshot (to locate the ··· icon)
Use computer tool: left_click on the ··· icon coordinates
```

3. In the dropdown menu, click **"Delete"**.

4. A confirmation modal appears with two options:
   - **"Delete list only"** — Removes the list but keeps all leads saved in your Sales Navigator
   - **"Delete list and unsave [X] leads"** — Removes the list AND unsaves all leads from it

5. Select the appropriate radio button, then click the **"Delete list"** button to confirm.

```javascript
// Select "Delete list and unsave" option (if desired)
const unsaveRadio = Array.from(document.querySelectorAll('input[type="radio"]'))
  .find(r => r.closest('label')?.textContent?.includes('unsave'));
if (unsaveRadio) unsaveRadio.click();

// Click the Delete list button
const deleteBtn = Array.from(document.querySelectorAll('button'))
  .find(b => b.textContent?.trim() === 'Delete list');
if (deleteBtn) deleteBtn.click();
```

**Important:** Always confirm with the user before deleting a list, especially the "unsave leads" option which is irreversible.

---

## Screenshot Policy

**Screenshots are PROHIBITED during normal operation.** Never take a screenshot to "check" the page, verify a filter, confirm a result count, or see what happened after a click. All reading, clicking, and verification MUST be done via `javascript_tool`, `find`, `read_page`, `form_input`, and `computer` tools.

**The ONLY time a screenshot is permitted** is after **3 consecutive failed attempts** at the same action using DOM-based tools. In that case:
1. Take ONE screenshot to diagnose the issue
2. Use what you learn to fix the DOM query or approach
3. Resume operating via code immediately — do NOT take another screenshot

**Explicitly prohibited screenshot usage:**
- Do NOT screenshot to verify filters were applied — use `javascript_tool` to read filter pills
- Do NOT screenshot to check result counts — use `javascript_tool` to read the count
- Do NOT screenshot to see the page after navigation — use `javascript_tool` or `read_page`
- Do NOT screenshot between pages during multi-page saving
- Do NOT screenshot to verify a list was created — use `javascript_tool` to check for success messages
- Do NOT screenshot "just to be safe" or "to confirm" — trust the DOM

**Why this matters:** Each screenshot costs time and tokens. The DOM provides all the information needed. A full list-building session should use ZERO screenshots under normal conditions.

The key insight is that LinkedIn Sales Navigator uses an Ember.js framework, so:
- **Typeahead inputs require real keyboard events** — use the `computer` tool's `type` action, not JavaScript value setting
- **Buttons and checkboxes can be clicked via JavaScript** — `.click()` works on most interactive elements
- **Suggestions appear as `[role="option"]` elements** with Include/Exclude `[role="button"]` children
- **Modals use the `artdeco-modal` class** with standard input fields inside

## Error Handling

### Page redirect / Login required
- If Sales Navigator redirects to login or subscription page, stop and ask the user to log in manually
- Do not attempt to enter credentials

### Filter not found
- If an expand button for a filter isn't found, try scrolling the filter panel or clicking "See all filters"
- Some filters may only appear after "See all filters" is expanded

### Typeahead returns no suggestions
- The typed value may not match LinkedIn's taxonomy exactly
- Try shorter or more generic terms (e.g., "CEO" instead of "Chief Executive Officer")
- LinkedIn often uses the full formal title (e.g., "Chief Information Security Officer" not "CISO", "Vice President Security" not "VP of Security")
- If a term returns no results, try the full spelled-out version first, then abbreviations
- Clear the input field before typing a new term: use `computer` tool with `key` action "cmd+a" then type the new term
- Report to user and ask for an alternative term if nothing works

### Rate limiting / Captcha
- Stop immediately — do not attempt to bypass
- Report to the user and suggest waiting before retrying

### List creation fails
- Check if the list name is already taken
- Check if the user has reached the list limit (Sales Navigator has a cap on number of lists)
- Suggest a different name or ask the user to delete an old list
