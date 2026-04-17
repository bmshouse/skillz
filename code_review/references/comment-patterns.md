# Comment-Writing Patterns Reference

This reference covers the key frameworks for writing effective code review comments,
drawn from "Looks Good to Me: Constructive Code Reviews" (Braganza, 2025).

---

## What Makes a Comment Effective?

Effective comments share three characteristics:

1. **Objective** — Grounded in facts, not personal preference. Traceable to a standard,
   convention, ticket, or demonstrable problem.
2. **Specific** — Points to exact lines/methods/variables. Makes clear whether action is required.
3. **Outcome-focused** — Tells the author exactly what "done" looks like, using transformation
   verbs (rename, remove, consolidate, extract, fix, rewrite, move).

---

## The 5P Process (Before Writing Any Suggestion)

Before writing a suggestion comment, run through this mental checklist:

1. **Pause** — Stop before typing. Don't start the comment yet.
2. **Ponder** — Why do you want to make this suggestion? Is it necessary right now? Can you justify
   it with objective facts (a standard, a convention, a performance measurement)?
3. Then choose:
   - **Pass** — Can't justify it clearly? Don't add it to the review.
   - **Propose** — Can justify it? Write the comment with your full reasoning.
   - **Postpone** — Valid idea but out of scope? Note it for a future ticket or offline conversation.

The 5P process eliminates "review creep" — extra scope introduced during review that wasn't
part of the original task.

---

## Comment Signals (Severity Labels)

Prefix every comment with a signal so the author immediately knows what action is required:

| Signal | Meaning | Blocking by default? |
|--------|---------|---------------------|
| `needs change:` | Small-to-medium fix required; must be addressed | Yes |
| `needs rework:` | Major structural problem; initiate offline discussion | Yes |
| `align:` | Valid code, but violates team/repo conventions | Yes |
| `level up:` | Non-blocking improvement for a future PR | No |
| `nitpick:` | Purely subjective; never blocks a PR | No — never block |
| `praise:` | Something done well; no action needed | N/A |

---

## The Triple-R Pattern

For any comment requesting a change, structure it as:

- **Request** — One sentence describing what to do
- **Rationale** — One or two sentences explaining why (with links or references)
- **Result** — A measurable end state so the author knows when they're done

### Examples

**Moving a method:**
> `needs change: Can we move the AuthenticateUser() method into the AuthenticationUtilities library?
> Similar methods (ReauthenticateUser(), AuthenticateThirdPartyUser()) are already there, and it's
> called in more than three places. Our coding conventions say authentication logic belongs in that
> library [link]. After this change, all calls should route through the library rather than a
> standalone declaration.`

**Renaming a variable:**
> `needs change: Can we rename the variable "item" to something more descriptive?
> The name "item" is vague and loses the concept that this object represents a discount-eligible
> product. Unclear variable names make the function harder to maintain.
> Suggested names: "discountEligibleItem" or "discountEligibleProduct".`

**Replacing a pattern:**
> `needs change: Can we replace the map() on line 32 with a for loop?
> The map() is being used only for iteration — it generates a new array that's never used. A for
> loop makes the intent clearer and avoids the unnecessary allocation.
> After the change, line 32 iterates without creating a discarded array.`

**Security finding:**
> `needs change: Can we replace the string-concatenated SQL query on line 87 with a
> parameterized query?
> Concatenating user input directly into SQL creates a SQL injection vulnerability — an attacker
> could manipulate the input to extract or destroy data. PreparedStatement (Java) /
> SqlParameterCollection (.NET) / parameterized PDO (PHP) all provide safe alternatives.
> After the change, no user input should touch the SQL string directly.`

---

## Tone Rules — The Politeness Principles

Research shows that comments beginning with "you" are statistically more likely to be perceived
as hostile. Technical terms and first-person plural framing produce the best outcomes.

**Principle 1: Replace "you" with "we"**
- ❌ "You should move this class to a separate file."
- ✅ "Can we move this class to a separate file?"

Codebase quality is the whole team's responsibility — "we" reinforces that, even when only the
author will do the work.

**Principle 2: Ask, don't command**
- ❌ "Move the Vehicle class to a separate file."
- ✅ "Can we move the Vehicle class to a separate file? [+ rationale]"

**When you're unsure:** Ask yourself these questions before posting:
- Do I mention the person more than the code?
- Is this polite and outcome-focused?
- Could any part be misread as combative or passive-aggressive?
- Will this make sense to a future developer reading the thread cold?

---

## Handling Disagreements: The MMG Exchange

When you and the author genuinely disagree on clarity or approach (not just style), use the
Maintainable Middle Ground (MMG) Exchange process:

1. Keep tone professional and respectful, even in disagreement
2. Acknowledge the concern; invite a discussion
3. Each side explains their reasoning and goals
4. Try small modifications first: align with existing conventions, add an explanatory comment, or add docs
5. If still no consensus, look for middle ground (combine elements; do a mini pair-programming session)
6. If still stuck, bring in the team
7. The solution that's clearest to the majority of developers — including future ones — wins

The goal is never to win — it's to find what's most maintainable for the team long-term.

---

## Alternative Commenting Systems

### MoSCoW Comments
A prioritization system from project management, adapted for code review:

| Label | Meaning |
|-------|---------|
| `M:` (Must) | Required change; blocks approval |
| `S:` (Should) | Clear improvement; author needs a reason to skip it |
| `C:` (Could) | Nice to have; above and beyond; ignorable |
| `W:` (Would) | Personal preference only; 100% ignorable |

### Conventional Comments (conventionalcomments.org)
A standardized format: `<label> [decorations]: <subject>` followed by optional `[discussion]`

Labels: `suggestion:`, `issue:`, `praise:`, `todo:`, `question:`, `thought:`, `chore:`, `note:`, `nitpick:`

Decorations: `(blocking)`, `(non-blocking)`, `(if-minor)`, `(security)`, `(ux)`, `(backend)`

---

## Code Compliments

When something is genuinely clever, elegant, or teaches you something, say so. A `praise:` comment:
- Reinforces good patterns for the whole team
- Counterbalances the inherently critical nature of review
- Builds psychological safety

Save compliments for things that are truly notable — not everything. For junior developers
and interns, be more generous: positive reinforcement builds confidence and teaches by example.

Examples:
> `praise: Smart use of memoization here — this would have been expensive to recalculate on every render.`
> `praise: I wasn't aware of this utility function. Excellent reuse.`
