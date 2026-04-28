---
name: git-commit-helper
description: Generate conventional commit messages with bilingual (English+Chinese) content for staged git changes. Use whenever the user asks to commit, generate a commit message, amend a commit message, or says "提交", "commit", "提交信息". Also trigger when the user provides bug/task links (PMS, Bugzilla, Jira) along with a commit request. Covers both new commits and `--amend` operations.
---

# Git Commit Helper

You generate structured, bilingual git commit messages following the company's conventional commit format. The goal is to produce clear, concise messages that other developers can quickly understand — while consuming minimal tokens during analysis.

## Workflow

### 1. Gather context

Determine what the user wants:

- **New commit**: Analyze staged files and generate a message
- **Amend**: Rewrite the last commit's message (preserving existing metadata)
- **User hints**: The user may provide inline instructions, bug links, task links, or suggestions — treat these as high-priority input

Collect any links the user provides (PMS tasks, Bugzilla bugs, Jira issues, etc.). These go into the footer.

### 2. Analyze staged changes (token-efficiently)

Run `git diff --cached --stat` first to get the file list. This is your primary analysis tool — cheap and informative.

**Smart file handling:**
- Read `git diff --cached` for source code files (.cpp, .h, .qml, .js, .py, .cmake, .pro, .pri, etc.)
- For translation files (.ts, .po, .pot, .json locale files), only check the stat — do NOT read their full diff. Summarize as "Updated translations in X language(s)"
- For generated files, lock files, binary files — skip entirely, just note they were changed
- If the diff is very large (>500 lines), read only the first 200 lines and summarize the rest from the stat

**What to extract:**
- What files changed and their purpose (from file paths)
- What functional changes were made (from the diff content)
- The nature of the change: fix, feat, refactor, chore, docs, style, etc.

### 3. Generate the commit message

The message MUST follow this exact structure:

```
<type>(<scope>): <english title>

1. <English bullet point 1>
2. <English bullet point 2>
3. <English bullet point 3>
...

Log: <One-line English summary of the key change>

<type>(<scope>): <chinese title>

1. <中文要点 1>
2. <中文要点 2>
3. <中文要点 3>
...

Log: <一行中文总结关键变更>
PMS: <TYPE>-<ID>
```

**Parts explained:**

**Title (line 1):** Conventional commit format: `type(scope): description`
- Types: `fix`, `feat`, `refactor`, `chore`, `docs`, `style`, `perf`, `test`, `build`, `ci`
- Scope: the module or component affected (e.g., `user-license`, `login`, `cmake`)
- Description: imperative mood, no period, lowercase start
- English title and Chinese title should convey the same meaning

**Content (numbered list):** What changed and why, in bullet points.
- Each point should be one clear, concise sentence
- Focus on the WHAT and WHY, not just file-level "changed file X"
- 3-6 points is ideal — don't pad, don't be overly verbose
- English section: pure English
- Chinese section: pure Chinese, mirrors the English points

**Log (summary line):** One sentence summarizing the key change in plain language.
- English Log mirrors the English content
- Chinese Log mirrors the Chinese content
- This should answer: "What does this commit do in one sentence?"

**Footer (links):** Bug/task links, immediately after Chinese Log (no blank line).
- Format: `PMS: <TYPE>-<ID>` (e.g. `PMS: BUG-358691`, `PMS: TASK-388139`, `PMS: STORY-123`)
- **URL parsing rules for PMS:**
  - `https://pms.uniontech.com/bug-view-358691.html` → `PMS: BUG-358691`
  - `https://pms.uniontech.com/task-view-388139.html` → `PMS: TASK-388139`
  - `https://pms.uniontech.com/story-view-123.html` → `PMS: STORY-123`
  - Extract TYPE from URL path: `bug-view`, `task-view`, `story-view`, etc.
- For other systems: `Bug: 12345`, `Jira: PROJ-789`
- Multiple links: one per line
- Do NOT write full URLs — use the shorthand format
- When amending, preserve all existing links from the original message

### 4. Handle amend

When the user asks to amend the last commit message:

1. Read the current message: `git log -1 --format=%B`
2. Extract existing footer links (PMS, Bug, Jira, Change-Id, etc.) — these MUST be preserved
3. Extract existing Log: lines — keep as reference but regenerate if the user provides new hints
4. Re-analyze the staged changes (if any new ones) using the same token-efficient approach
5. Generate the new message, carrying over all preserved metadata
6. The user can still provide hints/prompts that override the original content

**Preserved metadata during amend:** Links, Change-Id, PMS IDs, any footer fields. These are never lost.

### 5. Preview and confirm

Always show the final commit message to the user before executing. Format it clearly with code fences:

````
Here's the generated commit message:

```
<full message here>
```

Does this look good? I'll commit after you confirm.
````

Only run `git commit` or `git commit --amend` after the user explicitly confirms.

### 6. User-provided hints

The user may provide hints when requesting a commit, such as:
- "focus on the performance improvement"
- "the scope is login-module"
- "bug link is 12345"
- "change type to refactor"
- Or just free-text context about what the commit does

Integrate these hints at high priority — they override any analysis-based inference. If the user's hint conflicts with what the diff shows, follow the user's hint.

The user may also provide context through the conversation — e.g., they've been explaining a bug fix and then say "ok commit it". Use the conversation context to inform the message.

## Format rules

- Title line: `type(scope): description` — no trailing period
- Content bullets: numbered `1. 2. 3.` — one per change
- Log: plain one-liner, no markup
- Footer: `PMS: <TYPE>-<ID>` — immediately after Chinese Log with NO blank line
- Blank line between each major section (title/content/log) except footer
- The entire message is two blocks (English then Chinese) separated by a blank line
- Keep lines under 100 characters where possible — wrap naturally if needed
- Don't be verbose — other devs scan these quickly

## Token-saving tips

- `git diff --cached --stat` before `git diff --cached` — skip if stat tells you enough
- Never read translation diffs fully
- Summarize, don't quote — you're explaining changes, not reproducing them
- If the user provides enough context, you may not need to read the diff at all — just ask to confirm
