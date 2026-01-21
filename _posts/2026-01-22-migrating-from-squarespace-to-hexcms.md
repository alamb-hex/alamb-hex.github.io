---
layout: post
title: "Migrating From Squarespace to HexCMS: Building the Tools You Need"
date: 2026-01-22
categories: [development, cms, tools]
tags: [hexcms, squarespace, migration, nodejs, automation, markdown]
author: Aaron Lamb
description: "Building an automated migration tool to move blogs from Squarespace to HexCMS. Three migration strategies, image handling, and lessons learned from migrating a real blog with 60+ posts."
---

# Migrating From Squarespace to HexCMS: Building the Tools You Need

I built HexCMS because I wanted a Git-based CMS without the security headaches of WordPress or the complexity of enterprise solutions. ([Full story here](/development/cms/security/2026/01/20/building-hexcms-security-simplicity.html))

Then I had to actually use it.

A client runs a hobbyist blog with 60+ posts and hundreds of images. They've been on Squarespace for years. Works fine, but they're paying monthly for features they don't use and dealing with customization limitations.

"Can you move it to HexCMS?"

Sure. How hard could migrating a blog be?

## The Squarespace Export Problem

Squarespace has an export feature. It gives you an XML file. I downloaded it, opened it, immediately regretted the decision.

**What you get:**
- Malformed XML (WordPress export format, sort of)
- HTML content that's half Squarespace's custom blocks, half inline styles
- Image URLs that point to Squarespace CDN
- No clear structure for metadata
- Tags and categories mixed together randomly
- Dates in weird formats

**What you need for HexCMS:**
- Clean Markdown files
- YAML frontmatter (title, date, author, tags)
- Local images downloaded and properly referenced
- Consistent slug format
- Proper directory structure

The XML export was useless. I'd have to scrape the live site.

## Three Migration Strategies

I built three approaches because different situations need different tools.

### Strategy 1: RSS Feed Migration

Squarespace provides RSS feeds. Quick, easy, reliable. One problem: RSS feeds only include the most recent 25 posts.

The blog has 60+ posts. RSS wouldn't cut it.

But RSS is perfect for testing. Fast feedback loop. Parse 25 posts, check the output, iterate quickly.

```bash
npm run migrate -- --dry-run
```

This became my development workflow. Make changes, run RSS migration in dry-run mode, check markdown output, repeat.

### Strategy 2: Sitemap Scraping

For complete migrations, I needed the sitemap.

Squarespace generates a `sitemap.xml` with every blog post URL. Parse the sitemap, scrape each URL, extract content and images, convert to Markdown.

Challenges:
- Rate limiting (don't hammer the server)
- Different Squarespace templates use different HTML structures
- Images embedded in various formats
- Need to respect robots.txt (even though it's our own site)

Implementation:
```typescript
// Polite scraping: 1.5 second delay between requests
// Try multiple content selectors (different templates)
// Exponential backoff on failures
// Detailed logging for debugging
```

Works perfectly for full migrations. Takes time, but gets everything.

### Strategy 3: Single Post Migration

Sometimes you just need to migrate one post. Test the migration. Fix a broken import. Migrate new content.

```bash
npm run migrate:sitemap -- --url=https://www.example-blog.com/blog/my-post
```

Same scraping logic as sitemap mode, but targets one URL. Fast iteration for debugging specific posts.

## The Image Problem

Every blog post has 5-15 images. Photos, diagrams, reference materials. Squarespace hosts them on their CDN.

**Challenge 1: Image download failures**

Network issues. Squarespace rate limiting. Large images timing out. Random 500 errors from their CDN.

Solution: Retry logic with exponential backoff. Failed image? Wait 1 second, retry. Failed again? Wait 2 seconds. Then 4, 8, 16, max 30 seconds.

```typescript
// Download with retry
// Track failures
// Log which images failed for manual recovery
// Continue migration even if some images fail
```

Most failures are temporary. Retry logic caught 95% of them. The remaining 5% logged to console for manual download.

**Challenge 2: Concurrent downloads**

60 posts × 10 images = 600 image downloads. Sequential downloads would take hours.

Concurrent downloads: 5 at a time. Fast enough. Not so fast that we overwhelm Squarespace's servers or my network.

Configurable via `imageConcurrency` setting. Slow network? Set to 2. Fast connection? Set to 10.

**Challenge 3: Image organization**

Where do images go? One giant folder? Post-specific directories?

Decision: Post-specific directories. Each post gets an `images/blog/post-slug/` folder.

Why:
- Clean organization (find images by post name)
- No naming conflicts (two posts can have "cover.jpg")
- Easy to delete post and all its images together
- Matches HexCMS conventions

**Challenge 4: Image references in Markdown**

Original Squarespace HTML:
```html
<img src="https://images.squarespace-cdn.com/content/v1/abc123/def456/image.jpg" />
```

Converted Markdown needs:
```markdown
![Image description](/images/blog/post-slug/image.jpg)
```

The migrator:
1. Extracts image URL from HTML
2. Downloads to local directory
3. Generates local path
4. Converts HTML `<img>` to Markdown with correct path
5. All automatic during conversion

## HTML to Markdown Conversion

Squarespace stores content as HTML with custom styling and classes. HexCMS wants clean Markdown.

Used Turndown library. Works great for basic HTML. Had to customize for Squarespace quirks.

**Problem 1: Nested formatting**

Squarespace loves nested `<div>` wrappers with inline styles. Turndown preserves the nesting, output looks terrible.

Solution: Strip Squarespace-specific classes before conversion. Let Turndown handle clean HTML.

**Problem 2: Custom blocks**

Squarespace has custom blocks for galleries, quotes, call-outs. These render as complex HTML structures.

Solution: Custom Turndown rules for common patterns. Image galleries become lists of images. Block quotes get cleaned up. Code blocks preserve formatting.

**Problem 3: Embedded content**

YouTube embeds. Twitter embeds. Instagram posts. Squarespace wraps these in `<iframe>` tags with their own styling.

Solution: Preserve the embed codes but clean up the wrapper. Markdown doesn't have native embeds, but Next.js can process them.

## Frontmatter Generation

Every HexCMS post needs YAML frontmatter:

```yaml
---
title: "Post Title"
author: "Blog Author"
publishedAt: "2024-01-15"
excerpt: "First 150 characters of post..."
featuredImage: "/images/blog/post-slug/cover.jpg"
status: "published"
featured: false
tags: ["hobby", "topic-1", "topic-2"]
---
```

Squarespace provides most of this data, just in different formats.

**Title:** Clean from HTML `<h1>` or meta tags.

**Author:** Squarespace doesn't expose author in RSS or sitemap. Set a default author name in config. Configurable per blog.

**Date:** Extract from URL slug or meta tags. Squarespace uses consistent date format in permalinks.

**Excerpt:** First paragraph of content, stripped of HTML, truncated to 150 chars.

**Featured Image:** First image in post content becomes featured image. Download to `cover.jpg`.

**Tags:** Squarespace mixes tags and categories. Extract both, deduplicate, lowercase for consistency.

## Incremental Migration

First migration run: 60 posts. Takes 15 minutes. Works perfectly.

Then I found a bug in image path generation. Fixed it. Now what? Re-migrate all 60 posts?

Added `skipSlugs` configuration. List of post slugs already migrated. Migrator skips them.

```javascript
skipSlugs: [
  'already-migrated-post',
  'another-old-post',
]
```

Incremental migrations:
1. Migrate initial batch
2. Find issues in output
3. Fix migrator code
4. Add migrated slugs to `skipSlugs`
5. Re-run migration (only processes new posts)
6. Repeat until perfect

For this migration, I ran 4 passes. First got 60 posts. Second fixed image paths (skipped 60, migrated 0). Third caught 3 new posts published during migration (skipped 60, migrated 3). Fourth was final verification.

## Real-World Numbers

**Client blog migration:**
- 63 total posts
- 487 images downloaded
- 15 minutes total runtime
- 0 manual interventions after setup
- 2 image download failures (recovered via retry logic)

**Output:**
- 63 clean Markdown files
- YAML frontmatter formatted correctly
- All images local and referenced properly
- Ready for HexCMS without modifications

**Comparison to manual migration:**
- Manual: 5-10 minutes per post × 63 posts = 5-10 hours
- Automated: 10 minutes setup + 15 minutes migration = 25 minutes
- Time saved: 4.5-9.5 hours

## What I Learned: Migrator Edition

**Dry run mode is critical.**

Can't overstate this. Migrations are destructive. You don't want to discover bugs after writing 63 files. Dry run mode shows exactly what will happen without changing anything.

I ran dry-run mode probably 50 times during development. Caught issues early. Validated fixes immediately.

**Multiple strategies beat one perfect solution.**

RSS migration is fast but incomplete. Sitemap migration is complete but slow. Single-post migration is perfect for debugging.

Don't force users into one approach. Give them options. Let them choose based on their situation.

**Logging is your debugging tool.**

Network failures. HTML parsing issues. Image download problems. You can't predict every edge case.

Verbose logging mode saved me hours. When something failed, logs told me exactly where and why.

```bash
npm run migrate:sitemap -- --verbose
```

Shows every step: fetching sitemap, parsing URLs, scraping content, downloading images, writing files. Critical for debugging production migrations.

**Retry logic pays off.**

Network requests fail. Servers have bad moments. Timeouts happen.

Exponential backoff with max retries solved 95% of failures automatically. The 5% that failed logged clearly for manual recovery.

**Parallel downloads with limits.**

Sequential: Too slow. Unlimited parallel: Overwhelms server.

5 concurrent downloads hit the sweet spot. Fast enough. Polite enough. Configurable for different scenarios.

**Configuration files beat hard-coded values.**

Every migration is different. Different blog URL, different output directory, different author name, different concurrency needs.

`migrator.config.js` makes the tool reusable. Took 10 extra minutes to implement. Saved hours on subsequent uses.

## When You Need This

**You're migrating from Squarespace to anything Markdown-based:**

HexCMS, Jekyll, Hugo, Gatsby, Next.js with MDX, Astro. If your target needs Markdown and local images, this tool works.

**You have a large blog:**

Small blog (5-10 posts)? Manually copy content. Large blog (50+ posts)? Automation pays off.

**You value your time:**

Manual migration is tedious. Copy content, download images, format frontmatter, fix references, repeat 50 times.

Automated migration: Configure once, run, verify. Use your time for customization and design instead.

**You want repeatability:**

Migrating multiple blogs with similar structure? Configure once, reuse everywhere.

I can now migrate any Squarespace blog to HexCMS in under 30 minutes. Most of that is configuration and verification, not actual work.

## What's Next for the Tool

**Current state:**

Built specifically for Squarespace to HexCMS. Works reliably for that use case. The patterns (retry logic, multi-strategy approach, incremental migration) should work for other Markdown-based targets, but I haven't tested them yet.

If you need help migrating off Squarespace, reach out: [Hexaxia contact form](https://www.hexaxia.tech/contact) or [LinkedIn](https://www.linkedin.com/company/hexaxia-technologies-llc).

**Future improvements:**

Better error recovery (resume from last successful post instead of re-processing).

Support for Squarespace's newer block formats (they keep changing HTML structure).

Video migration (currently skips embedded videos, could download them).

Export validation (verify every image downloaded, every link works, every frontmatter field populated).

**But honestly:**

It works well enough. The client blog migrated perfectly. Tool is reliable for its purpose. Additional features would be nice-to-have, not critical.

Software doesn't need to be perfect. It needs to solve the problem. This solves the problem.

## The Bigger Picture

This migrator exists because I built HexCMS. HexCMS exists because I wanted Git-based content management without WordPress security headaches.

Building your own tools creates a cascade of related tools. CMS requires a migrator. Migrator requires error handling. Error handling requires logging. Each piece enables the next.

I spent maybe 8 hours building this migrator. Saved 5-10 hours on the first migration. Will save another 5-10 hours on the next migration. ROI is already positive.

More importantly: I control the entire stack. Content in Git. CMS serves from Git. Migrator populates Git. No vendor lock-in anywhere. No services that can be deprecated. No APIs that can change.

**That's the real win.**

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development.

**Project Links:**
- [HexCMS Architecture Post](/development/cms/security/2026/01/20/building-hexcms-security-simplicity.html)
- Need migration help? [Contact Hexaxia](https://www.hexaxia.tech/contact)
