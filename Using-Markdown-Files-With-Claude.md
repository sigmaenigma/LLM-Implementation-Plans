# Using Markdown Plans with Claude Code

A practical guide to how I use markdown files to get better results when building features with AI coding tools like Claude Code in VS Code.

---

## The Core Idea

Claude Code doesn't carry memory between sessions. Markdown files committed to your repo act as persistent context — a breadcrumb trail that tells Claude (and future you) what was built, why, and what's left to do.

## The Workflow

### 1. Start With an Implementation Plan

Before writing code, have Claude help you draft a plan as a markdown file. A good plan includes:

- A clear description of the feature
- Broken-down steps with checkboxes (`- [ ]` / `- [x]`)
- Any dependencies, edge cases, or constraints
- Acceptance criteria

Save it somewhere like `docs/plans/001-user-auth.md`.

### 2. Generate Test Cases From the Plan

Once the plan feels solid, ask Claude to create test case scenarios based on it. This does two things:

- **Validates the plan** — if Claude can't write meaningful tests from it, the plan is too vague. Tighten it up before writing any code.
- **Builds a regression safety net** — you have tests in place before you start layering on features.

### 3. Mark Progress as You Go

As steps get completed, update the checkboxes. When the feature is done, add a note at the top or bottom:

```
> ✅ **This feature has been implemented.**
```

This makes it immediately obvious when someone (or Claude) opens the file later.

### 4. Commit Everything to Version Control

This is the step most people skip, and it's what makes the whole system compound over time. Committed plans let you do things like:

> "Hey, here's the doc from when we built the auth system. We just added role-based permissions — can you update this document to reflect the new features and think through any ramifications or issues this could cause with the current implementation?"

Claude is strong at this kind of impact analysis when it has the original context to work from.

### 5. Reuse Plans as Living Documentation

These files double as documentation after the fact. You don't have to reconstruct what happened from chat logs — the plan, the test cases, and the status markers are all right there in your repo.

---

## Why This Works

| Problem | How plans solve it |
|---|---|
| Claude forgets everything between sessions | The markdown file *is* the memory |
| Features break when you add new ones | Test cases generated from the plan catch regressions |
| Nobody documents anything | The plan *is* the documentation — it already exists |
| "What were we thinking when we built this?" | `git log` on the plan file shows the full evolution |

---

## Tips

- **Use a naming convention early.** Something like `docs/plans/001-feature-name.md` keeps things organized. Without it, you end up with `plan-v2-final-FINAL.md` pretty quickly.
- **Ask Claude to think through ramifications.** When adding to an existing feature, point Claude at the original plan and ask it to consider what might break or need updating. This is one of its stronger use cases.
- **Keep plans scoped.** One feature per file. If a plan is covering three unrelated things, split it up.
- **Don't delete old plans.** Even fully implemented ones are useful as context for future changes.

---

## Example Prompts

When starting a feature:
> "I want to add user authentication with email/password. Can you create an implementation plan as a markdown file with checkboxes for each step?"

When generating tests:
> "Based on this implementation plan, create test case scenarios that cover the happy path and edge cases."

When evolving a feature:
> "Here's the plan from when we built auth. We're adding OAuth support now. Update this document to reflect the new features and flag anything that might conflict with the current implementation."

When checking in after a break:
> "Read through the plan files in `docs/plans/` and tell me where we left off."
