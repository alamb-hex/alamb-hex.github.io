---
layout: post
title: "When Your AI Assistant Wastes 9 Attempts: Building Better Interfaces"
date: 2026-01-20
categories: [development, ai, tooling]
tags: [ai, efficiency, automation, claude, rag, developer-experience]
author: Aaron Lamb
description: "How a simple wrapper script eliminated 90% of AI command failures. A practical case study in building AI-friendly tooling."
---

# When Your AI Assistant Wastes 9 Attempts: Building Better Interfaces

I use Claude Code heavily for technical work. It's great at architecture, debugging, writing code. But I noticed a pattern: certain tasks resulted in multiple failed attempts before success.

**The symptom:** 9 consecutive errors trying to do one simple thing.

**The diagnosis:** My tools weren't designed for AI collaboration.

**The fix:** 10 minutes of wrapper scripting eliminated the entire problem class.

## The Problem: 9 Failed Attempts

I asked Claude to add some documents to my RAG (Retrieval Augmented Generation) system. Simple task. Here's what happened:

```bash
# Attempt 1-3: Wrong Python command
python episodic_memory.py --add file.md
# Error: command not found

# Attempt 4-6: Missing virtual environment
python3 episodic_memory.py --add file.md
# Error: ModuleNotFoundError: No module named 'chromadb'

# Attempt 7-9: Wrong script entirely
source venv/bin/activate && python episodic_memory.py --add file.md
# Error: unrecognized arguments: --add
```

**The actual command needed:**
```bash
cd .claude/rag && source venv/bin/activate && \
  python embed_documents.py --files ../../output/file.md
```

Claude eventually got there, but wasted significant time (and my API budget) failing first.

## Why This Happens

AI assistants are excellent at:
- Understanding intent
- Writing new code
- Debugging logical errors
- Following patterns

AI assistants struggle with:
- **Remembering exact CLI syntax** (which vs. where)
- **Environment setup** (is it `python` or `python3`?)
- **Subtle distinctions** (two scripts with similar names, different args)
- **Undocumented conventions** (must cd to specific directory first)

The issue isn't the AI. **The issue is my interface was designed for humans who learn once, not AI that starts fresh every conversation.**

## The Root Cause: Two Similar Scripts

My RAG system had two Python scripts:

**`episodic_memory.py`** - Stores conversation history
```bash
python episodic_memory.py --store \
  --state "User asked X" \
  --action "Did Y" \
  --outcome "Result Z"
```

**`embed_documents.py`** - Embeds documents for search
```bash
python embed_documents.py --files path/to/file.md
```

Both lived in `.claude/rag/`, both required venv activation, both processed markdown files. **Claude couldn't reliably pick the right one.**

Add in:
- Ubuntu's `python` vs `python3` quirk
- Virtual environment activation requirement
- Different working directory expectations
- Similar but incompatible argument syntax

You get 9 failed attempts.

## The Solution: Single Entry Point

I created a simple wrapper script:

```bash
#!/bin/bash
# ./sage-rag - Single entry point for all RAG operations

VENV_PYTHON="$SCRIPT_DIR/venv/bin/python3"

case "${1:-help}" in
    embed)
        shift
        "$VENV_PYTHON" "$SCRIPT_DIR/embed_documents.py" "$@"
        ;;

    episode)
        shift
        "$VENV_PYTHON" "$SCRIPT_DIR/episodic_memory.py" "$@"
        ;;

    search)
        shift
        "$VENV_PYTHON" "$SCRIPT_DIR/search.py" "$@"
        ;;

    health)
        "$VENV_PYTHON" "$SCRIPT_DIR/embed_documents.py" --health-check
        ;;

    *)
        echo "Usage: ./sage-rag [embed|episode|search|health]"
        ;;
esac
```

**New interface:**
```bash
./sage-rag embed --files output/file.md
./sage-rag episode --list
./sage-rag search "query"
./sage-rag health
```

**What this eliminates:**
- ❌ No more `python` vs `python3` confusion
- ❌ No more venv activation required
- ❌ No more choosing between scripts
- ❌ No more working directory issues
- ❌ No more syntax guessing

**Result:** Zero failed attempts since implementation.

## The Documentation Fix

I also added a quick reference file the AI reads first:

**`QUICK-REF.md`:**
```markdown
# Quick Reference for Claude

**Use the wrapper script `./sage-rag` instead of calling Python directly.**

## Common Operations

### Embed Documents
```bash
cd .claude/rag && ./sage-rag embed --files ../../output/filename.md
```

### Search
```bash
cd .claude/rag && ./sage-rag search "your query"
```

## Common Mistakes to Avoid

1. **Don't call Python directly** - use the wrapper
2. **Don't mix up the two systems** - documents ≠ episodes
3. **Don't try "add" command** - uses `--store`, not `add`
```

Now when Claude needs to use RAG:
1. Reads `QUICK-REF.md` (3 seconds)
2. Uses correct command (first try)
3. Moves on

## Broader Lessons: AI-Friendly Tooling

This pattern applies beyond RAG systems. If you're building tools that AI assistants will use:

### 1. Single Entry Points Beat Multiple Scripts

**Instead of:**
```bash
python analyze_data.py --input file.csv
python transform_data.py --input file.csv --output transformed.csv
python validate_data.py transformed.csv
```

**Use:**
```bash
./data-tools analyze file.csv
./data-tools transform file.csv -o transformed.csv
./data-tools validate transformed.csv
```

### 2. Consistent Argument Patterns

AI struggles with subtle variations:
- `--file` vs `--files` vs `--input`
- `--state` vs `--status` vs `--condition`

Pick one convention. Stick to it everywhere.

### 3. Self-Documenting Help Text

```bash
./tool help
./tool [command] --help
```

Should show:
- Available commands
- Common examples
- Expected argument format
- Most common mistakes

### 4. Environment Abstraction

Don't make the AI remember:
- Virtual environment activation
- Working directory requirements
- PATH setup
- Environment variables

Wrapper handles all of it.

### 5. Fail Fast with Clear Errors

**Bad error:**
```
Error: unrecognized arguments
```

**Good error:**
```
Error: unrecognized command 'add'
Did you mean: --store (for episodes) or embed (for documents)?
See ./sage-rag help
```

## Implementation Checklist

Building an AI-friendly CLI wrapper:

- [ ] Single executable entry point
- [ ] Subcommands instead of separate scripts
- [ ] Handle environment setup internally
- [ ] Consistent argument naming across commands
- [ ] Built-in help text with examples
- [ ] Quick reference doc (QUICK-REF.md)
- [ ] Clear error messages with suggestions
- [ ] Version checking/health commands

**Time investment:** 30 minutes to build wrapper + docs

**Time saved:** Hours of debugging + reduced API costs

## The Tools Addition

After fixing the wrapper issue, I asked what system packages would help. The AI suggested:

- **`fd-find`** - Better file finding (clearer than `find`)
- **`yq`** - YAML parsing (like `jq` for YAML)
- **`bat`** - Syntax highlighting

These aren't for the AI—they're for making bash commands more readable and reliable. Better tooling means fewer edge cases the AI has to handle.

**Installing them was trivial. Not installing them would be penny-wise, pound-foolish.**

## Results

**Before wrapper:**
- 9 failed attempts to embed 3 files
- ~2 minutes wasted per RAG operation
- Frequent context switching to debug

**After wrapper:**
- 0 failed attempts (100+ operations since)
- <5 seconds per operation
- No debugging needed

**ROI:**
- 30 minutes to build
- Saved 2+ hours in first week
- Eliminated entire error class

## Takeaway

When you see your AI assistant making the same mistakes repeatedly, **the problem isn't the AI**. The problem is your interface wasn't designed for AI collaboration.

**Simple wrappers, clear docs, consistent patterns**—these aren't "nice to have." They're force multipliers for AI-assisted development.

The best part? These same improvements make your tools better for humans too.

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development.
