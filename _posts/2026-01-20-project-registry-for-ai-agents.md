---
layout: post
title: "The One File That Stopped My AI Agent From Asking 'Which Project?'"
date: 2026-01-20
categories: [ai, automation, productivity]
tags: [ai, context, rag, agents, efficiency, knowledge-management]
author: Aaron Lamb
description: "How a simple structured registry eliminated context confusion across 30+ projects and made my AI assistant actually useful."
---

# The One File That Stopped My AI Agent From Asking 'Which Project?'

When you work across 30+ projects, your AI agent's biggest problem isn't intelligence. It's context.

Sage, my AI executive assistant, kept asking the same questions: "Which project is this for?" "Where does that file live?" "Is ULS a ham radio thing or a customer?"

That last one actually happened. Sage confused United Life Services (a client paying $3,600/month) with the FCC's Universal Licensing System.

Not because it was stupid. Because the context was scattered across memory files, old notes, and my head.

**The fix:** A single structured file that serves as the authoritative source of project truth.

## The Problem: Context Confusion

Here's what a typical interaction looked like before:

> **Me:** Check the contract status
> **Sage:** Which contract? I see references to KE-GPT, Pax Nocturna, and several others.
> **Me:** KE
> **Sage:** Got it. Looking at ke-agent...
> **Me:** No, the contract is in the company folder, not the code folder.

This happened constantly. Every task required 2-3 rounds of clarification. The friction added up fast.

## The Solution: Project Registry

The Project Registry is a structured markdown file at `.claude/memory/projects.md`. It contains everything Sage needs to understand my project landscape.

### Project Types Matter

Not all projects are the same. A client company is different from a codebase is different from a marketing site.

I defined six types:

| Type | Description |
|------|-------------|
| **Company** | Client/customer entity (contracts, proposals, business docs) |
| **Code** | Software development project (source code, deployable) |
| **Site** | Website or marketing site |
| **Product** | Internal product under development |
| **Research** | R&D and experimental projects |
| **Personal** | Personal projects and planning |

This distinction is critical. When I say "check the KE files," Sage now knows to look in the company folder for contracts, not the code folder for source files.

### Owner Mapping

Every project has an owner or client. The registry maps this explicitly:

| Owner | Company Folder | Code Projects |
|-------|----------------|---------------|
| **KE Construction** | ke-construction | KE-Agent, KE-GPT |
| **ULS** | uls | (none) |
| **Hexaxia** | hexaxia-internal | Hextant, AgentForge, HexCMS |

Now when someone mentions "Jay's project," Sage immediately knows that means KE Construction, which has both company folders and code projects.

### Relationship Mapping

Projects don't exist in isolation. They depend on each other, share technology, and serve business hierarchies.

**Dependencies:**
```
Ham Radio Today → HexCMS (content management)
Lyfe Uncharted → HexCMS (content management)
Sage → APF (operational protocols)
```

**Business Hierarchy:**
```
Hexaxia Technologies (parent)
├── Hexaxia AI (division)
│   ├── Hextant (product)
│   ├── AgentForge (product)
│   └── hexaxia.ai (marketing site)
├── Hexaxia Media (division)
│   └── TT Carnival 2026 (first project)
└── Websites
    ├── hexaxia.tech (main company)
    └── hexaxia.com (splash)
```

**Shared Tech Stacks:**

| Stack | Projects |
|-------|----------|
| Next.js 16 + Tailwind v4 | DDTS, KE-Agent, HexCMS Studio |
| Next.js 15 + HexCMS | Ham Radio Today, Lyfe Uncharted |
| Next.js + Supabase | AgentForge, KE-Agent |

Understanding these relationships means Sage can apply fixes to shared dependencies, understand upgrade impacts, and route questions correctly.

## How It Works

The registry is embedded into my RAG (Retrieval Augmented Generation) system using local Ollama embeddings. When I ask Sage a question, it semantically searches the registry for relevant context before responding.

**Example workflow:**

1. I say: "What's the status on Jay's project?"
2. Sage searches RAG, finds "Jay" associated with "KE Construction"
3. Registry shows: company folder + code project paths
4. Sage reads the relevant files and gives an accurate status update

**After the registry:**

> **Me:** Check the KE contract status
> **Sage:** Contract v0.9.1 is drafted, awaiting signatures. $12k total, 20/40/40 payment structure.

That's the difference between an assistant and a friction generator.

## Technical Implementation

**Embedding Strategy:**
- 1000 character chunks with 200 character overlap
- Sentence-aware splitting keeps logical units together
- HTTP API for embeddings (not CLI) avoids parsing bugs
- Local Ollama (nomic-embed-text model) for zero API costs

**Performance:**
- Total embedding time: ~17 seconds for 17 chunks
- Search latency: <150ms
- Cost: $0 (100% local)

**Maintenance:**
Adding a new project takes 30 seconds:
1. Add entry to appropriate section
2. Add to Quick Reference table
3. Update relationships if applicable
4. Re-embed

Compare this to training the agent through conversation, which is slow, inconsistent, and doesn't persist across sessions.

## Results

Since implementing the project registry:

- **Zero project confusion errors** in two weeks
- **Faster responses** - no clarifying questions needed
- **Better task routing** between company folders and code projects
- **Easier onboarding** when resuming work after breaks

The registry also serves as documentation for me. When I forget which client uses which tech stack, I check the registry instead of digging through project folders.

## Key Lessons

### 1. Structure Beats Volume

A well-structured 300-line registry beats a 3000-line brain dump. The relationships table alone saves more time than pages of prose descriptions.

### 2. Types Are Essential

The company/code/site distinction wasn't obvious at first. But an AI agent needs to know the difference between a folder of contracts and a folder of source code.

### 3. Relationships Are High Value

The dependency graph and business hierarchy took 10 minutes to write but provide disproportionate value. When Sage understands organizational structure, it navigates conversations without getting confused.

### 4. Embed Locally

Using local embeddings (Ollama) instead of API-based embeddings means:
- Zero marginal cost for updates
- No rate limits
- Data never leaves the machine
- Works offline

The quality tradeoff is minimal for this use case. We're matching project names and relationships, not doing nuanced semantic analysis.

## Pattern for Your Team

The Project Registry pattern is reusable. Any team working with AI agents across multiple projects, clients, or domains would benefit from a similar approach.

**Minimum viable registry:**

1. **Project Types** - Define 4-6 types relevant to your work
2. **Owner Mapping** - Who owns what, where it lives
3. **Quick Reference Table** - Name, type, path, status
4. **Relationships** - Dependencies and hierarchies
5. **Embed** - Make it searchable via RAG

Start simple. Add complexity only when you hit friction.

## The Real Insight

AI agents are only as good as their context. You can have the most capable model in the world, but if it doesn't know which project you're talking about, it will waste your time asking clarifying questions or make mistakes that cost more time to fix.

The smarter the agent, the more it suffers from context gaps.

By building a structured, searchable, authoritative source of project truth, we turned Sage from a capable but confused assistant into one that actually knows what I'm talking about.

**The best part:** This solves a human problem too. The registry is now my go-to reference when I need to remember project details.

Good tools make both humans and AI more effective.

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development. He builds AI-powered tools for MSPs and SMBs, including Hextant (virtual C-suite) and AgentForge (white-label agent framework).
