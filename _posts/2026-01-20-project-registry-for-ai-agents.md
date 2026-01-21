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

my AI executive assistant kept asking the same questions: "Which project is this for?" "Where does that file live?" "Is XYZ the client company or the internal project?"

That last one actually happened. my AI assistant confused a high-value client with an internal research initiative that shared the same acronym.

Not because it was stupid. Because the context was scattered across memory files, old notes, and my head.

**The fix:** A single structured file that serves as the authoritative source of project truth.

## The Problem: Context Confusion

Here's what a typical interaction looked like before:

> **Me:** Check the contract status
> **my AI assistant:** Which contract? I see references to Client-A, Project-B, and several others.
> **Me:** Client A
> **my AI assistant:** Got it. Looking at client-a-app...
> **Me:** No, the contract is in the company folder, not the code folder.

This happened constantly. Every task required 2-3 rounds of clarification. The friction added up fast.

## The Solution: Project Registry

The Project Registry is a structured markdown file at `.ai/memory/projects.md`. It contains everything my AI assistant needs to understand my project landscape.

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

This distinction is critical. When I say "check the Client A files," my AI assistant now knows to look in the company folder for contracts, not the code folder for source files.

### Owner Mapping

Every project has an owner or client. The registry maps this explicitly:

| Owner | Company Folder | Code Projects |
|-------|----------------|---------------|
| **Client A** | client-a | ClientApp, ClientGPT |
| **Client B** | client-b | (none) |
| **Internal** | company-internal | ProductX, ProductY, CMS |

Now when someone mentions a key contact's project, my AI assistant immediately knows which company it refers to, and whether it has both company folders (contracts) and code projects.

### Relationship Mapping

Projects don't exist in isolation. They depend on each other, share technology, and serve business hierarchies.

**Dependencies:**
```
Blog A → CMS (content management)
Blog B → CMS (content management)
AI Assistant → Framework (operational protocols)
```

**Business Hierarchy:**
```
Company (parent)
├── AI Division
│   ├── Product X (product)
│   ├── Product Y (product)
│   └── ai-site (marketing site)
├── Media Division
│   └── Event Project (first project)
└── Websites
    ├── main-site (main company)
    └── splash (splash)
```

**Shared Tech Stacks:**

| Stack | Projects |
|-------|----------|
| Next.js + Tailwind | Site A, ClientApp, CMS Studio |
| Next.js + CMS | Blog A, Blog B |
| Next.js + Database | ProductX, ClientApp |

Understanding these relationships means my AI assistant can apply fixes to shared dependencies, understand upgrade impacts, and route questions correctly.

## How It Works

The registry is embedded into my RAG (Retrieval Augmented Generation) system using local Ollama embeddings. When I ask my AI assistant a question, it semantically searches the registry for relevant context before responding.

**Example workflow:**

1. I say: "What's the status on the construction client's project?"
2. my AI assistant searches RAG, finds the contact associated with "Client A"
3. Registry shows: company folder + code project paths
4. my AI assistant reads the relevant files and gives an accurate status update

**After the registry:**

> **Me:** Check the Client A contract status
> **my AI assistant:** Contract v0.9.1 is drafted, awaiting signatures. $12k total, 20/40/40 payment structure.

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

The dependency graph and business hierarchy took 10 minutes to write but provide disproportionate value. When my AI assistant understands organizational structure, it navigates conversations without getting confused.

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

By building a structured, searchable, authoritative source of project truth, we turned my AI assistant from a capable but confused assistant into one that actually knows what I'm talking about.

**The best part:** This solves a human problem too. The registry is now my go-to reference when I need to remember project details.

Good tools make both humans and AI more effective.

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development.
