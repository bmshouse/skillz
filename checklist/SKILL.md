---
name: checklist
description: >
  Creates precise, professional checklists following the principles from
  "The Checklist Manifesto" by Atul Gawande. Use this skill whenever the
  user asks to make a checklist, build a procedure list, create a pre-flight
  check, design an SOP, draft a quality-control list, or document any
  step-by-step verification process — even if they just say "make me a
  checklist for X" or "what should I check before doing Y". The skill
  ensures every checklist is short, precise, and actually usable under
  pressure rather than being a bloated wall of text.
---

<!--
  Methodology sourced from:
  Gawande, Atul. "The Checklist Manifesto: How to Get Things Right."
  Picador, 2009.
-->

# Checklist Skill

Your job is to produce a checklist that is genuinely useful — one that
someone could pull out mid-task, under stress, and actually use. That
means short, precise, and focused on the steps that matter most.

## Step 1 — Clarify the context (if not already clear)

Before drafting, make sure you understand:

- **What is the task or situation?** (e.g., launching a product, prepping
  for a client call, deploying code, discharging a patient)
- **Who will use it?** (expert practitioner, novice, mixed team?)
- **What could go wrong if steps are skipped?** This surfaces the
  "killer items" — the things most dangerous to miss.

If the user's request is vague, ask one focused question to get what you
need. Don't interrogate them with a long list of questions.

## Step 2 — Decide the pause point

Every good checklist has a defined moment when it gets used. State this
clearly at the top of the checklist. Examples:

- "Before deployment begins"
- "Upon patient arrival in pre-op"
- "Before sending the proposal"

If it's obvious from context (like a warning light going on), you can skip
making it explicit — but usually naming the pause point prevents the
checklist from being ignored until it's too late.

## Step 3 — Choose the checklist type

Pick the type that fits how the work actually gets done:

**DO-CONFIRM** — The person does the tasks from memory and experience,
then pauses to confirm each item was completed. Best for experts who have
internalized the steps but need a safety net against skipping something
under pressure. (Pilots use this type.)

**READ-DO** — The person reads each item and performs it in sequence,
like following a recipe. Best for complex procedures where order matters,
or for less-experienced users.

State the type at the top of the checklist so it's unambiguous.

## Step 4 — Draft the items

Follow these rules ruthlessly:

**Keep it to 5–9 items.** If you feel you need more, you're probably
mixing two different checklists. Split them. A checklist that tries to
cover everything covers nothing — it just gets ignored.

**Focus on the killer items.** These are the steps that are: (a) critical
to safety or success, AND (b) most likely to be skipped or overlooked.
Routine steps that nobody ever forgets don't need to be on the list.
Steps that experts sometimes skip under pressure absolutely do.

**Use simple, exact language.** Write items as short, active phrases —
the way a professional in that field would actually say them. Not
"Ensure that the configuration file has been properly verified," but
"Config file verified." Use the vocabulary of the profession, not
generic bureaucratic wording.

**Be specific enough to be checkable.** Each item should have a clear
yes/no answer. Vague items ("everything looks good?") are useless because
they can always be answered yes.

## Step 5 — Format for use under pressure

A checklist that looks cluttered won't get used when things get hard.

- Fit it on **one page** (or one screen)
- Use **both uppercase and lowercase** (ALL CAPS is slower to read)
- Avoid unnecessary colors, decorations, or explanatory prose
- Use consistent, simple formatting — a plain checkbox list is ideal
- Group related items if the list has natural phases, but don't overdo it

## Output format

Produce the checklist as a clean, minimal markdown document:

```
PAUSE POINT: [When to use this checklist]
TYPE: [DO-CONFIRM or READ-DO]

CHECKLIST TITLE
───────────────────────────
[ ] Item one
[ ] Item two
[ ] Item three
...
```

Then, after the checklist, include a brief note (2–4 sentences) explaining:
- Why you chose DO-CONFIRM vs READ-DO
- Which items are the "killer items" and why
- Any item you deliberately left off and why

This transparency helps the user refine it — they may know things about
the situation that you don't.

## What makes a checklist bad (avoid these)

- **Too long.** If users see 20 items, they'll rush through or skip it
  entirely. Ruthlessly cut.
- **Vague items.** "Double-check everything" is not a checklist item.
- **Obvious steps.** Don't list things experts never forget. It's
  condescending and trains people to ignore the list.
- **No pause point.** A checklist with no defined trigger is just a
  document nobody reads.
- **Walls of explanation.** The checklist itself should be spare. Save
  explanations for a separate reference document.

## Iteration

After presenting the checklist, ask: "Does this match how the work
actually gets done — and are there any critical steps I missed or
obvious ones I should cut?" One round of feedback usually produces
something genuinely usable.
