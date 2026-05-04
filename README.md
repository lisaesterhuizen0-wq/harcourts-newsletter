# Harcourts Platinum: The Platinum Report

A monthly HTML email newsletter for **Harcourts Platinum**, a real estate agency
serving the Helderberg basin in South Africa.

**Live preview:** https://lisaesterhuizen0-wq.github.io/harcourts-newsletter/

## How it works

Each month, a Make.com scenario:

1. Pulls the agency's current featured listings.
2. Calls Claude, which runs a custom skill stored in my private Notion workspace.
   The skill generates the entire HTML newsletter (template, editorial copy,
   market commentary, and listing blurbs) from the listings data.
3. Pushes the generated HTML to this repo, served as a preview via GitHub Pages.
4. Emails the office principal a link to the preview, asking for approval
   before dispatch.

Final dispatch to the client list is handled separately, after the principal
signs off, keeping a human in the loop on every send.

## What's in this repo

- `index.html`: the most recent generated issue (Issue 4, April 2026), shown
  at the preview link above.

The Make.com scenarios and the Notion-stored Claude skill are not in this repo.

## Why it's interesting

The pattern is portable. A small agency can run a polished monthly newsletter
without an email-marketing platform. Claude does the writing, and a single
approval email keeps a human in control of what goes out. The same
generate → preview → approve flow adapts to any small business that wants
AI-drafted, human-approved outbound.

## Tech

- HTML + inline CSS (table-based layout, required by most email clients)
- Make.com (orchestration)
- Anthropic Claude (generation, via a custom Notion-stored skill)

---

*Part of my AI Operations portfolio (github.com/lisaesterhuizen0-wq)*
