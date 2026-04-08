# CLAUDE.md — Obsidian Ledger

This file is the source of truth for any AI agent working on this project.
Read it fully before writing a single line of code.

---

## Project Overview

**Obsidian Ledger** is a personal budget management app with a dark, glassmorphic
dashboard aesthetic.nees to be Android app
**Stack at a glance:**
- Frontend: react native
- Database: Supabase (PostgreSQL + Auth + RLS)
- Design: Dark glassmorphism, obsidian palette, no light mode


## Design System (Non-Negotiable)

### Color Palette (CSS Variables)
```css
--color-canvas:       #0e0e0e   /* page background */
--color-container:    #131313   /* low-level cards */
--color-surface:      #1a1a1a   /* elevated cards */
--color-accent:       #94aaff   /* primary accent */
--color-accent-dim:   #809bff   /* secondary accent */
--color-text-primary: #e8e8f0
--color-text-muted:   #8888a0
--color-border:       rgba(148, 170, 255, 0.15)  /* ghost border */
```