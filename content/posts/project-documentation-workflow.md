+++
date = '2026-08-15T12:00:00+07:00'
draft = false
title = 'Project Documentation: A Structured Notes Workflow'
description = "A systematic approach to documenting projects with daily logs, decision tracking, and task management"

categories = ["projects"]
tags = ["documentation", "workflow", "notes", "productivity", "project-management"]
authors = ["Experian"]
avatar = "/images/solder-profile-pic.webp"
+++

# Project Documentation: A Structured Notes Workflow

## Background

I have a bad habit of not documenting what I'm doing and what I've done properly. A lot of things have been lost even though I feel that I'm doing a lot every day. I struggle to tell people what I've actually done. The clearest example was when I tried to build this portfolio site, I hit a wall: the Home Automation project had generated a mountain of learnings over months, but most of it was lost because I hadn't captured it properly along the way.

Before this system, my project notes were whatever I felt like writing in the moment. A few scattered thoughts here, a half-finished todo list there. I told myself I'd come back and flesh it out. I never did. And when I needed to look back — like, right now, while writing this portfolio — it was blank. I knew I'd done good stuff, but the details were gone.

I tried kanban boards for organization. They help with tracking what needs to be done, but they're terrible at capturing how you figured it out, why you chose one approach over another, or the random insights that come at 2am. Then I built the SearXNG project with this structured workflow, complete with daily logs, decision tracking, everything that I can think of. And the difference was night and day. I no longer need to try to remember months of work — I just read what I'd already written. It was embarrassing. I felt stupid for not doing this earlier.

I must be honest, this works because I now have an AI assistant running locally on my machine. It really is making things easier.

## Overview

So what does the system actually look like? Everything lives in Markdown files inside a Notes vault on my machine. Each project is one file — or, once it gets big, a file plus a folder — and it tracks progress through daily logs, decision records, and a simple task board. It's just text files that any editor can open, and that my AI assistant can read and write just as easily. No external tools, no databases, and no cloud anything. 

I built it to solve exactly one problem: how do I capture what I know in a way that's useful both for me and for the agent, without letting it rot into a pile of unstructured notes nobody can navigate? Because I'd tried the unstructured route. It doesn't work.

## Structure

Every project file starts from the same template. I don't think about what shape to write anymore — I just fill it in:

```markdown
---
name: "Project Title"
type: coding | hardware | research | automation
status: To Do | In Progress | Blocked | Done | Paused
priority: high | medium | low
start-date: 2026-08-15
progress: 0
---

# Project Title

## Overview
## Goals
## Tasks
### To Do
### In Progress
### Blocked
### Done
## Daily Log
### 2026-08-15
- **Worked on:**
- **Breakthrough / Result:**
- **Learnings:**
- **Next Steps:**
## Key Decisions
| Decision | Date | Reason | Context |
## Resources
```

Projects come in two kinds:

- **Simple ones** — a single file, `Projects/Project Name.md`.
- **Complicated ones** — a folder with `main.md` at the center and whatever else it needs as siblings: runbooks, diagrams, extra docs.

The SearXNG project falls into the complicated kind. The runbook was a real output of the project — it documents how to operate the deployment — not a documentation file I created on the side. So it lives in `Projects/SearXNG Homelab Setup/main.md`, with a `runbook.md` sitting right next to it.

## The Daily Log Pattern

The heart of the whole thing is the Daily Log. Every entry has four parts: what I worked on, what came out of it — the breakthrough, the result, whatever it was — what I learned, and what I should do next. It's the difference between "I did stuff today" and "here's exactly what I did, why, and what it taught me."

When I log more than once in a day, the entries get numbered — entry 1, entry 2, entry 3. So the log just grows, and that rule matters: once something's written down, it stays written down.

### Decision Tracking

Whenever I make a key decision, I mark it with `[DECISION]` in the log. The system picks those up and lines them up in a table right after the daily log:

| Decision | Date | Reason | Context |
|----------|------|--------|---------|
| Deploy SearXNG in Docker on the homelab machine | 2026-08-02 | Standard, low-friction deployment | see 2026-08-02, entry 2 |

That table does two things for me:
1. **Quick reference** — scroll to the Key Decisions section and see every major call I've made at a glance.
2. **Reason preservation** — each decision keeps its *why* attached, not just the *what*. Six months later I don't have to guess why I chose something.

### Resource Auto-Capture

If a log entry has a URL in it, it gets picked up automatically and added to the Resources section with a short description of what it is. I don't bookmark anything manually anymore.

## How It Works in Practice

The whole workflow runs on plain English. No special syntax, nothing to memorize — I just talk to the assistant:

| Command | Action |
|---|---|
| "create a new project called X" | Creates a project file with the template |
| "Log this for X: today I figured out Y" | Appends a daily log entry, classified into the four sections |
| "mark task X as done" | Moves task between Kanban sections, checks the box |
| "mark project X as done" | Updates frontmatter status |
| "set progress to 75%" | Updates frontmatter progress |
| "add this link to X" | Appends resource to the project |

## Why This Works For Me

### It's just Markdown

No database, no sync client, and no proprietary format I'm locked into. The vault is a folder of files that anything can read — including me, and including the AI. If every tool I use died tomorrow, my notes would still be there, and they'd still be mine.

### It scales with complexity

Small projects stay small — just one file. Big ones get a folder with runbooks and diagrams, but the entry point always looks the same, so it reduces the cognitive load when I want to read it.

### It captures process, not just outcomes

Most project documentation tells you what got built. This captures how it got built — the decisions, the dead ends, the things I learned the hard way. That's where the real value is. The next project I work on gets to stand on that.
