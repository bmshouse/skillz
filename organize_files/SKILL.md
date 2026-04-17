---
name: organize-files
description: >
  Organizes a messy folder of files into a clean, actionable structure
  using the PARA method (Projects, Areas, Resources, Archives). Use this
  skill whenever the user asks to organize, clean up, sort, or restructure
  files or folders — whether it's their Downloads folder, a project
  directory, a desktop full of clutter, a shared drive, or any pile of
  digital files they can't find their way through. Also use it when the
  user says things like "my files are a mess," "I can't find anything,"
  "help me get organized," "sort through these documents," or "set up a
  folder structure for me." This skill doesn't just rename and sort — it
  builds a living system organized around what the user is actually working
  on right now, so files stay findable as life changes.
---

<!--
  Methodology sourced from:
  Forte, Tiago. "Building a Second Brain: A Proven Method to Organize
  Your Digital Life and Unlock Your Creative Potential."
  Atria Books / Simon & Schuster, 2022.
-->

# File Organization Skill

Your job is to turn a pile of files into a structure the user can actually
navigate and maintain — not a tidy archive that freezes the moment they
walk away from it.

The core principle: **organize by where files are going, not where they
came from.** A folder labeled "PDFs" or "Downloads" or "2023" tells you
nothing about what to do with the files inside. A folder labeled with the
project or responsibility it supports tells you exactly when to open it.

---

## Step 1 — Understand the user's situation

Before touching any files, ask the user two questions (combine them into
one message):

1. **What folder or location are we organizing?** (path, or ask them to
   share access if needed)
2. **What are you currently working on?** Ask them to name 3–7 active
   projects — things with a deadline or a specific outcome they're
   working toward right now. Examples: "launch the website," "finish
   the tax return," "plan the conference," "complete the course."

These active projects become the skeleton of the new structure. Without
knowing them, you'll build a tidy filing cabinet that has nothing to do
with the user's actual life.

---

## Step 2 — Detect the operating system

Before running any commands, detect which OS you're on:

```bash
uname -s 2>/dev/null || echo "Windows"
```

If `uname` is unavailable or returns nothing, you're on Windows and should
use PowerShell for all subsequent commands. On macOS/Linux, use bash.
If you already know from context (e.g., the path starts with `C:\`), skip
the check. Keep the detected OS in mind for every command in Steps 2–6.

---

## Step 3 — Survey the files

Once you have access, scan the target location.

**macOS / Linux (bash):**
```bash
find "<path>" -maxdepth 3 | head -200
```

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path "<path>" -Recurse -Depth 2 | Select-Object -First 200 FullName
```

Or list it at multiple levels to get a sense of what's there. You're
looking for:

- **Volume**: how many files and rough size
- **Age distribution**: are most files recent, or years old?
- **Types**: documents, images, code, PDFs, miscellaneous?
- **Existing structure**: any folders already in place?
- **Clear clusters**: groups of files that obviously belong together

Don't try to read every file name. Get the shape of the pile.

---

## Step 4 — Build the PARA structure

Create four top-level folders in the target location:

```
01 - Projects/
02 - Areas/
03 - Resources/
04 - Archive/
```

The numbers keep them sorted to the top in any file browser. Here's what
each one means:

### Projects — What you're working on right now
Short-term efforts with a specific outcome and a natural end point.
A project is done when the thing ships, launches, closes, or gets
checked off.

Create one subfolder per active project the user named in Step 1.
Examples:
```
01 - Projects/
  Website Redesign/
  Tax Return 2025/
  Conference Planning/
  Online Course - UX Design/
```

**The test for Projects**: Does it have an end? Will it eventually be
"done"? If yes, it's a project.

### Areas — What you're responsible for over time
Ongoing commitments with no finish line — just a standard you want
to maintain. These don't complete; they continue.

Create subfolders for the major areas of the user's life and work.
Common examples:
```
02 - Areas/
  Finances/
  Health/
  Career Development/
  Home/
  [Department or role at work]/
```

If the user hasn't told you their areas, use what you can infer from
the files themselves, and note what you've assumed so they can adjust.

**The test for Areas**: Is this an ongoing responsibility rather than
a project? Does it have a standard to maintain rather than a finish
line? If yes, it's an area.

### Resources — Things you might want later
Reference material on topics you find useful or interesting, not
tied to any current project or responsibility. Think of this as a
personal library.

Examples:
```
03 - Resources/
  Photography/
  Home Improvement Ideas/
  Reading Notes/
  Career - Job Postings/
  Recipes/
```

**The test for Resources**: Would you put this in a reference book or
a library rather than an inbox? Is it reference material rather than
active work? If yes, it's a resource.

### Archive — Everything else
Completed projects, old areas you're no longer responsible for,
outdated resources, and the entire pile of "old stuff" that existed
before this reorganization.

The Archive is not a trash can — nothing gets deleted. It's cold storage:
out of sight and off your mind, but always findable via search.

---

## Step 5 — Sort the files

Work through the files systematically. For each file or group of files,
ask one question:

> **Which of the user's active projects will benefit from this most?**

If one of their projects clearly benefits → move to that project folder.
If no project fits but it's an ongoing responsibility → move to the
relevant Area folder.
If it's general reference material → move to the relevant Resource folder.
If it's old, completed, or unclear → move to Archive.

**The decision order matters.** Always check Projects first, then Areas,
then Resources, then Archive. This keeps the most actionable files most
visible.

**The archive shortcut**: For large, old, chaotic collections, the fastest
path to a usable system is to archive everything that predates a certain
date (e.g., "older than 1 year"), then build the new structure fresh.
This is not losing anything — it's clearing the workspace. The user can
always retrieve archived files by searching.

Use this approach when:
- There are hundreds of unsorted files
- Most files are clearly old and not currently relevant
- The user's biggest problem is just finding current work

Ask the user's permission before doing a bulk archive like this.

---

## Step 6 — Handle the mechanics

When executing the moves, use commands appropriate to the detected OS.

### macOS / Linux (bash)

```bash
# Create the PARA structure
mkdir -p "<path>/01 - Projects"
mkdir -p "<path>/02 - Areas"
mkdir -p "<path>/03 - Resources"
mkdir -p "<path>/04 - Archive"

# Create a project subfolder
mkdir -p "<path>/01 - Projects/Project Name Here"

# Move a file or folder (skip if destination exists)
mv -n "<source>" "<destination>"
```

Bulk archive (get user permission first — moves all unsorted root items):
```bash
find "<path>" -maxdepth 1 -not -name "0*" -not -name "." \
  -exec mv -n {} "<path>/04 - Archive/" \;
```

### Windows (PowerShell)

```powershell
# Create the PARA structure
$base = "<path>"
New-Item -ItemType Directory -Force -Path "$base\01 - Projects"
New-Item -ItemType Directory -Force -Path "$base\02 - Areas"
New-Item -ItemType Directory -Force -Path "$base\03 - Resources"
New-Item -ItemType Directory -Force -Path "$base\04 - Archive"

# Create a project subfolder
New-Item -ItemType Directory -Force -Path "$base\01 - Projects\Project Name Here"

# Move a file or folder (skip if destination already exists)
if (-not (Test-Path "<destination>")) {
    Move-Item -Path "<source>" -Destination "<destination>"
}
```

Bulk archive (get user permission first — moves all unsorted root items):
```powershell
$base = "<path>"
Get-ChildItem -Path $base -Depth 0 |
  Where-Object { $_.Name -notlike "0*" } |
  ForEach-Object {
    $dest = "$base\04 - Archive\$($_.Name)"
    if (-not (Test-Path $dest)) { Move-Item $_.FullName $dest }
  }
```

Before running any bulk moves, show the user what will happen and confirm.

---

## Step 7 — Show the result

After organizing, print a directory tree (depth-limited for readability).

**macOS / Linux:**
```bash
find "<path>" -maxdepth 3 -type d | sort | sed 's|[^/]*/|  |g'
```

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path "<path>" -Recurse -Depth 2 -Directory |
  Select-Object FullName | Sort-Object FullName
```

If `tree` is available on either platform, it gives a nicer view:
```bash
tree "<path>" -d -L 3          # macOS/Linux (may need: brew install tree)
```
```powershell
tree "<path>" /F /A             # Windows (built-in)
```

---

Present the result to the user with a brief explanation:
- How many files were organized
- Where everything landed
- What was archived vs. kept active
- Any assumptions you made that they should verify

---

## Step 8 — Leave them with a simple maintenance rule

Good organization doesn't require constant upkeep. It just requires
one habit: **when you start a new project, make a folder for it first.**

That single discipline — creating a home before you create the content —
is what makes the whole system self-maintaining.

Also tell the user:
- When a project finishes, move its folder from Projects → Archive
- When a new ongoing responsibility appears, add it to Areas
- When in doubt about where to put something, put it in the most
  relevant *active project*. Better to be slightly wrong and findable
  than perfectly categorized and buried.

---

## What not to do

**Don't organize by file type.** "PDFs," "Images," "Word Docs" as
top-level categories mean nothing about what to do with the files.
They're organized by where they came from, not where they're going.

**Don't organize by date alone.** "2022," "2023," "2024" is slightly
better than nothing, but still doesn't help you find what you need
for the project you're working on today.

**Don't over-engineer.** Three levels of nesting (PARA category →
project/area → file) is almost always enough. Deep hierarchies get
abandoned. Flat, labeled buckets get used.

**Don't wait until it's perfect.** An imperfect structure you actually
use beats a beautiful system you're still building. Move quickly,
touch lightly. Files can always be moved again.

**Don't delete things.** When uncertain, archive. The goal is a clear
workspace, not an empty one.
