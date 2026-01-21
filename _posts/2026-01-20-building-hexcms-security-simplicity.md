---
layout: post
title: "Building HexCMS: Security and Simplicity Through Git"
date: 2026-01-20
categories: [development, cms, security]
tags: [hexcms, git, security, cms, markdown, open-source]
author: Aaron Lamb
description: "Why I built a Git-based CMS from scratch focused on security and simplicity, and why I had to build a visual editor to complete the vision."
---

# Building HexCMS: Security and Simplicity Through Git

I built HexCMS because every CMS I evaluated had the same fundamental problem: **the more features they added, the more attack surface they created**. WordPress, Drupal, even modern headless CMSs - they all prioritize flexibility over security.

I wanted the opposite: **security first, simplicity always**.

## The Core Problem with Traditional CMSs

Here's what bothered me about existing solutions:

**WordPress:** Great ecosystem, terrible security model. Plugins can do anything. Database gets compromised, entire site is gone. Updates break things constantly.

**Headless CMSs (Contentful, Strapi, etc.):** Better than WordPress, but still require authentication, API keys, admin panels. Every feature is another potential vulnerability. Most are overkill for a simple blog.

**Static Site Generators (Jekyll, Hugo):** Close to what I wanted, but non-technical users can't use them. Writing in Git directly isn't realistic for most content creators.

## Why I Built This

At Hexaxia, we build AI platforms and solve complex infrastructure challenges for clients. Our marketing sites needed secure, simple content management. Rather than compromise our security standards with WordPress or accept the complexity of enterprise CMSs, we built our own. HexCMS now powers our blog, documentation, and client sites.

## The HexCMS Approach: Git as the Source of Truth

I made one architectural decision that solved multiple problems: **Content lives in Git. Everything else is derived.**

**Two deployment modes:**

### Mode 1: Git-Only (Pure Simplicity)

Perfect for small to medium blogs:
1. Write Markdown in Git (any repo, any branch)
2. Push to GitHub/GitLab
3. Next.js reads directly from Git and serves content via ISR
4. No database required

### Mode 2: Git + PostgreSQL (Scale)

For large content sets that create build bottlenecks:
1. New Markdown post added to Git
2. Webhook triggers sync to PostgreSQL
3. Next.js reads from database for faster builds
4. Git remains the source of truth (database is a synced cache)

**Why this matters:**
- **Start simple, scale when needed** - Git-only until you hit build bottlenecks
- **Provider flexibility** - Supabase (full feature set), Vercel's Neon (simple, low-mid tier)
- **Git is always authoritative** - database sync failures just rebuild from Git
- **No lock-in** - can switch between modes without losing content

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
- **Works with any Git workflow** - branches, PRs, reviews - all standard Git operations
- **Platform agnostic** - GitHub, GitLab, Gitea, self-hosted - doesn't matter

## The Problem I Didn't Anticipate

HexCMS worked exactly as designed: secure, simple, Git-based. But it had one fatal flaw for real-world adoption:

**Non-technical users struggled with raw Markdown.**

Even developers who understood Git found the pure Markdown workflow cumbersome. My wife wanted to write blog posts for Lyfe Uncharted - asking her to write in raw Markdown syntax, remember frontmatter formatting, commit with descriptive messages, and push to main was not realistic.

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

My wife can now write blog posts in a visual editor, click "Publish," and the content goes live - without me having to manage an admin panel, authentication system, or security updates.

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

## Future Enhancements

Some challenges I'm working through as HexCMS matures:

**Multi-author workflows:** Git handles branches and PRs well, but non-technical users struggle with merge conflicts. Exploring ways to make collaborative editing feel natural without compromising the Git-first model.

**Image optimization:** Currently images go through Git (which works), but optimizing them at build time would reduce bandwidth. Investigating where that logic lives without compromising the Git-first approach.

**Real-time preview:** HexCMS Studio shows Markdown preview, but not the actual rendered site. A local Next.js preview would be ideal - balancing the complexity tradeoff.

## What's Next

HexCMS is currently in beta testing internally at Hexaxia and powering our production sites. We're refining the workflow and hardening edge cases before a public release.

If you're building a blog or documentation site and want:
- Security without complexity
- Git-based workflow
- No admin panel to maintain
- Simple deployment (Next.js + Git, optionally + PostgreSQL)
- Start simple, add complexity only when needed

HexCMS might be worth exploring once we go public.

**Status:** Beta (internal testing)
**Code:** GitHub repos coming with public release
**Stack:** Node.js, Git webhooks, Next.js, PostgreSQL (optional)

**Security note:** HexCMS leverages Next.js security best practices for API routes and data fetching. I'll cover the specific security architecture in a future post on hardening Next.js applications.

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development.
