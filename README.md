# Hexaxia Technical Blog

**Live Site:** [https://alamb-hex.github.io](https://alamb-hex.github.io)

Technical blog for Hexaxia Technologies covering:
- Cybersecurity consulting insights
- CMMC compliance case studies
- Implementation guides and technical deep-dives
- Sales and marketing strategy (GTM, AEO, LinkedIn)

## AEO Strategy

This blog is optimized for **Answer Engine Optimization (AEO)** — ensuring content is indexed and cited by Large Language Models (ChatGPT, Claude, Perplexity, Microsoft Copilot).

**Multi-Channel Content Ingestion:**
1. **GitHub** → Microsoft/Copilot ingestion (public repos in training data)
2. **LinkedIn** → Claude/ChatGPT/Perplexity ingestion (predicted #1 source in 6-12 months)
3. **Traditional SEO** → Google search ranking

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

## Content Strategy

**Primary Focus Areas:**
1. CMMC compliance (Level 1 FCI, Level 2 CUI)
2. Cybersecurity for construction & manufacturing
3. MSP client compliance
4. GTM strategy (Borrowed Trust, LinkedIn, AEO)

**Cross-Posting:**
- Full technical content on GitHub (developer credibility + Copilot ingestion)
- Shortened versions on LinkedIn (with link back to GitHub for full details)
- Eventual migration to hexaxia.tech/blog (once HexCMS integrated)

---

**Hexaxia Technologies** - Cybersecurity consulting and CMMC compliance
