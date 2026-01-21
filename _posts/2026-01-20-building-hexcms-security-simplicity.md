---
layout: post
title: "Building HexCMS: When Security and Simplicity Aren't Enough"
date: 2026-01-20
categories: [development, cms, security]
tags: [hexcms, git, security, cms, markdown, open-source]
author: Aaron Lamb
description: "Why I built a Git-based CMS from scratch focused on security and simplicity, and why I had to build a visual editor to complete the vision."
---

# Building HexCMS: When Security and Simplicity Aren't Enough

I built HexCMS because every CMS I evaluated had the same fundamental problem: **the more features they added, the more attack surface they created**. WordPress, Drupal, even modern headless CMSs—they all prioritize flexibility over security.

I wanted the opposite: **security first, simplicity always**.

## The Core Problem with Traditional CMSs

Here's what bothered me about existing solutions:

**WordPress:** Great ecosystem, terrible security model. Plugins can do anything. Database gets compromised, entire site is gone. Updates break things constantly.

**Headless CMSs (Contentful, Strapi, etc.):** Better than WordPress, but still require authentication, API keys, admin panels. Every feature is another potential vulnerability. Most are overkill for a simple blog.

**Static Site Generators (Jekyll, Hugo):** Close to what I wanted, but non-technical users can't use them. Writing in Git directly isn't realistic for most content creators.

## The HexCMS Approach: Git as the Source of Truth

I made one architectural decision that solved multiple problems: **Content lives in Git. Everything else is derived.**

Here's the core workflow:
1. Write Markdown in Git (any repo, any branch)
2. Push to GitHub/GitLab
3. Next.js reads from Git and serves content via ISR

**That's it.** No database required.

But if you need performance at scale, HexCMS supports an **optional PostgreSQL cache**:
- Webhook triggers sync to PostgreSQL
- Next.js reads from database instead of Git
- Faster queries for large sites (100+ posts)
- Database is just a cache—Git is still the source of truth

**Why this works for security:**

- **No admin panel to hack** - there's no login, no dashboard, no attack surface
- **Version control is built in** - every change is tracked, auditable, reversible
- **Database is optional** - start with pure Git, add PostgreSQL only if you need the performance
- **Separation of concerns** - content storage (Git) is separate from content delivery (Next.js)
- **Stateless by design** - if using a database, it's just a cache; Git is always the source of truth
- **No file uploads** - Markdown and assets go through Git, which has its own security model

**Why this works for simplicity:**

- **Start with zero infrastructure** - just Git + Next.js, no database to set up
- **One source of truth** - Git is always correct; database sync failures just rebuild from Git
- **No migrations** - content is Markdown files, not database schemas
- **Progressive complexity** - add the database only when you need it, not from day one
- **Works with any Git workflow** - branches, PRs, reviews—all standard Git operations
- **Platform agnostic** - GitHub, GitLab, Gitea, self-hosted—doesn't matter

## Real-World Usage

HexCMS currently powers two live sites, showing both deployment modes:

**Ham Radio Today** ([hamradiotoday.vercel.app](https://hamradiotoday.vercel.app))
- 42+ posts migrated from Squarespace
- Two authors (David and I)
- **Uses PostgreSQL** (Neon) for fast queries across large content set
- Zero security incidents since launch

**Lyfe Uncharted** ([lyfe-uncharted.vercel.app](https://lyfe-uncharted.vercel.app))
- Personal travel blog
- Migrated from Squarespace
- **Pure Git mode** (no database) - simple deployment, fewer moving parts
- Dark theme, gold accents, handwritten fonts
- Caribbean sailing/cruising content

Both sites run without admin panels, without authentication systems, without vulnerability scanners flagging issues. The database decision was purely about performance needs, not architecture requirements.

## The Problem I Didn't Anticipate

HexCMS worked exactly as designed: secure, simple, Git-based. But it had one fatal flaw for real-world adoption:

**Non-technical users can't write in Git.**

My wife wanted to write blog posts for Lyfe Uncharted. She's not a developer. Telling her "just edit the Markdown file in VS Code, commit with a descriptive message, and push to main" was... not realistic.

I needed a content editor. But I refused to compromise on the core principles:
- **No web-based admin panel** (introduces attack surface)
- **No authentication system** (security complexity)
- **Git must remain the source of truth** (no database mutations)

## Enter HexCMS Studio

HexCMS Studio is a **local desktop application** (Electron-style Next.js app) that makes HexCMS usable for non-developers while preserving the security model.

**What it does:**
- **WYSIWYG editor** (Tiptap) + **code mode** (CodeMirror 6) for Markdown
- **Frontmatter form editor** - fill in title, date, tags via a clean form
- **Git integration** - stage, commit, push, pull directly from the UI
- **Multi-repo support** - manage multiple HexCMS sites from one app
- **Multi-theme** - Light, Dark, Midnight, Sepia themes for different preferences

**Why it works:**
- **Runs locally only** - requires filesystem access, never exposed to the internet
- **No cloud dependencies** - edits happen on your machine, push when ready
- **Git workflow preserved** - commits go to Git, not a database
- **Zero server-side code** - it's just a Markdown editor with Git commands

My wife can now write blog posts in a visual editor, click "Publish," and the content goes live—without me having to manage an admin panel, authentication system, or security updates.

## The Design Philosophy

Building HexCMS taught me that **simplicity is a security feature**.

Every feature you add to a CMS is:
- Another thing that can break
- Another attack vector to defend
- Another migration to manage
- Another permission model to audit

By refusing to add features that compromise the core Git-based model, HexCMS stays small, auditable, and secure.

**What HexCMS doesn't have (intentionally):**
- User authentication
- Role-based access control
- Media library with uploads
- Plugin system
- Database migrations for content
- Admin dashboard
- API keys or tokens

**What it does have:**
- Git as the source of truth
- Webhook-triggered sync
- PostgreSQL cache for fast reads
- Next.js ISR for CDN-friendly delivery
- Full content versioning (via Git)

## Lessons Learned

**1. "Secure by default" means removing features, not adding them.**

I spent more time deciding what NOT to build than what to build. Every feature request got filtered through: "Does this compromise Git as the source of truth?"

**2. The best admin panel is no admin panel.**

HexCMS Studio proved that you can have a great editing experience without a web-based admin system. Running locally eliminates entire categories of vulnerabilities.

**3. Simplicity scales better than flexibility.**

HexCMS can't do everything WordPress can do. That's the point. It does one thing well: serve Markdown content from Git. That constraint makes it reliable.

**4. Two tools are sometimes better than one.**

I initially resisted building HexCMS Studio. "Just use Git!" But separating the CMS (server-side) from the editor (client-side) actually made both simpler. HexCMS stayed focused on content delivery. Studio stayed focused on content creation.

## Open Questions

I'm still working through some challenges:

**Multi-author workflows:** Git handles branches and PRs well, but non-technical users struggle with merge conflicts. How do you make collaborative editing feel natural without a web-based draft system?

**Image optimization:** Currently images go through Git (which works), but optimizing them at build time would be better. Where does that logic live without compromising the Git-first model?

**Real-time preview:** HexCMS Studio shows Markdown preview, but it's not the actual rendered site. A local Next.js preview would be ideal, but adds complexity.

## What's Next

HexCMS is in production and working well for my use cases. The next step is seeing if other developers find value in this approach.

If you're building a blog or documentation site and want:
- Security without complexity
- Git-based workflow
- No admin panel to maintain
- Simple deployment (Next.js + Git, optionally + PostgreSQL)
- Start simple, add complexity only when needed

HexCMS might be worth exploring.

**Code:** Available on GitHub (HexCMS core + HexCMS Studio repos)
**Live examples:** Ham Radio Today (with DB), Lyfe Uncharted (pure Git)
**Stack:** Node.js, Git webhooks, Next.js, PostgreSQL (optional)

---

**About the Author:** Aaron Lamb builds security-focused tools and writes about cybersecurity consulting, CMMC compliance, and bootstrap GTM strategy. Currently working on making Git-based content management accessible to non-developers.
