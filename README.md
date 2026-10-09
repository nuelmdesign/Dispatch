# Dispatch

A personal email studio for designed email: newsletters, announcements, reminders and celebrations, sent to a chosen group in a few clicks.

## How this repo works

- **CLAUDE.md** is the build guide. Claude Code reads it at the start of every session. It holds the product scope, tech stack, design rules and the build order.
- **Build in slices.** Each slice in section 15 of CLAUDE.md gets its own branch (for example `slice-0-setup`). When a slice works, merge it into `main`. That way `main` is always a working version.
- **reference/** holds screenshots from Bunce and Flodesk for Claude Code to study before the design-heavy slices.
- **.env.example** lists the secret settings the app needs. Copy it to `.env.local` and fill it in. Never commit `.env.local`.

## Starting a session

Open Claude Code on this repo and the current slice branch, then say:

> Read CLAUDE.md fully, including the opendraft rules it links to. Then write your plan for the current slice and wait for my go. Explain anything technical in plain English.

## How to run locally

Claude Code will fill this in during slice 0.
