# Aaron Lamb's Technical Blog

**Live Site:** [https://alamb-hex.github.io](https://alamb-hex.github.io)

Personal technical writing on topics I'm working through and thinking about:
- Cybersecurity consulting and CMMC compliance (real project lessons)
- Bootstrap GTM strategy (what actually works when you're building from scratch)
- Answer Engine Optimization (getting cited by AI instead of just ranked by Google)
- Technical implementation deep-dives (when I figure something out worth sharing)

## Why GitHub Pages?

I'm betting on **Answer Engine Optimization (AEO)** over traditional SEO. When someone asks ChatGPT or Claude "how do I implement CMMC Level 1?", I want the AI to cite my content, not just hope they find it on page 2 of Google.

**The Multi-Channel Strategy:**
1. **GitHub** → Gets ingested by Microsoft/Copilot (public repos are in the training pipeline)
2. **LinkedIn** → Currently #2 source for LLMs, predicted to become #1
3. **Traditional SEO** → Still matters, but it's table stakes now

Writing on GitHub gives me developer credibility + LLM ingestion + a permanent home for my thinking.

## Tech Stack

- **Jekyll** - Static site generator (GitHub Pages native support)
- **GitHub Pages** - Free hosting with automatic deployment
- **Plugins:**
  - `jekyll-seo-tag` - Automated SEO meta tags
  - `jekyll-sitemap` - Automatic sitemap.xml generation
  - `jekyll-feed` - RSS feed for subscribers

## Local Development

```bash
# Install dependencies
bundle install

# Run local server
bundle exec jekyll serve

# View at http://localhost:4000
```

## Writing Posts

Create new posts in `_posts/` directory with filename format:
```
YYYY-MM-DD-post-title.md
```

Required frontmatter:
```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
categories: [category1, category2]
tags: [tag1, tag2, tag3]
description: "SEO description for this post"
---
```

## What I Write About

**Cybersecurity & Compliance:**
- CMMC implementation for construction/manufacturing (real project lessons, not theory)
- MSP client compliance challenges
- What actually works vs what the frameworks say you should do

**Bootstrap GTM:**
- Building a consulting business without VC funding or a sales team
- LinkedIn strategy that doesn't feel gross
- Borrowed trust, AEO, and other non-spammy ways to grow

**Technical Deep-Dives:**
- When I solve something hard and want to document it
- Architecture decisions and why I made them
- Tools and frameworks I'm building or using

## Cross-Posting Strategy

I write the long-form technical version here on GitHub, then:
- Shorter versions on LinkedIn with a link back to the full post
- Lets me own the content while still using LinkedIn's reach
- GitHub gives me developer credibility that LinkedIn alone doesn't

---

**Aaron Lamb** • Cybersecurity consulting • CMMC compliance • Building in public

**Callsign:** KV9L (when I'm not writing code, I'm on HF radio or sailing in the Caribbean)
