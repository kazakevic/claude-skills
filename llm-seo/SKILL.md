---
name: llm-seo
description: "Optimize content and websites for AI search visibility across ChatGPT, Perplexity, Google AI Overviews, Gemini, and Claude. Use when the user wants to: (1) optimize existing content for LLM citation, (2) audit AI search visibility, (3) create or review llms.txt files, (4) generate schema markup for AI discoverability, (5) structure content using chunk-based architecture for RAG retrieval, (6) develop GEO/AEO/LLMO strategy, (7) write content designed to be cited by AI systems, (8) analyze competitors' AI search presence, (9) create FAQ structures optimized for AI extraction. Also use when user mentions 'LLM SEO', 'GEO', 'AEO', 'LLMO', 'AI search optimization', 'AI Overviews optimization', 'Perplexity ranking', 'ChatGPT citation', 'generative engine optimization', 'answer engine optimization', or 'AI visibility'."
---

# LLM SEO Optimization Skill (2026)

## Core Concept

LLM search engines use RAG (Retrieval-Augmented Generation): they retrieve **chunks** (self-contained fragments), assemble them as context, then generate a synthesized answer with inline citations. Optimization means making your content the chunk that gets retrieved and cited.

**Optimal chunk size:** 134–167 words for Google AI Overviews; 62% of featured content falls in the 100–300 word range. General safe range: 75–225 words.

**Critical positioning insight:** 44.2% of all LLM citations come from the first 30% of text (the intro). Lead every page and section with the most citable content.

This is NOT a replacement for traditional SEO — it's a complement. Google sends 345x more traffic than all AI platforms combined. 76% of AI Overview citations also rank in Google's top 10.

**The AI learner test:** Ask yourself — *"If an AI had to learn this topic using only my page, could it confidently explain it to someone else?"* If yes, you're doing LLM SEO correctly. This is a better proxy than any technical checklist.

**Core framework:** Think in terms of **Clarity → Structure → Authority → Coverage** — not just metadata.

## Task Router

Determine the user's task and follow the appropriate workflow:

**Content optimization?** → Follow "Content Optimization Workflow" below
**Website/technical audit?** → Follow "Technical Audit Workflow" below
**llms.txt creation?** → Follow "llms.txt Generation" below
**Schema markup?** → Follow "Schema Generation" below
**Strategy/planning?** → Consult [references/strategy-guide.md](references/strategy-guide.md)
**Content writing?** → Follow "AI-Optimized Content Writing" below

## Content Optimization Workflow

When optimizing existing content for LLM visibility:

1. **Restructure into chunks** — Each H2/H3 section = one idea, 100–300 words, self-contained answer (optimal 134–167 words for AI Overviews)
2. **Front-load the intro** — 44.2% of all LLM citations come from the first 30% of text. Put your most citable content first
3. **Answer-first formatting** — Lead every section with the key takeaway, then context. AI may only extract first 2-3 sentences
4. **Add verifiable facts** — Statistics with attribution, named studies, specific data. AI cites content it can cross-reference
5. **Entity-based language** — Write about concepts and relationships, not keyword phrases. LLMs identify entities and map them (e.g. Bitcoin → cryptocurrency, market cap → financial metric, exchange → trading venue). Pages that explicitly describe these relationships are favored. Avoid stuffing: instead of "bitcoin price bitcoin price bitcoin price", write "Bitcoin price reflects current market demand, trading volume, liquidity, and macroeconomic factors such as inflation and interest rates."
6. **FAQ sections** — Add 4-6 natural-language questions as H2/H3 headings with concise answers beneath
7. **Freshness signals** — Add explicit "Last updated: [date]" near top of content. Pages updated within 3 months get ~67% more citations
8. **Strong lead sentences** — Each paragraph starts with a quotable, clear point
9. **Length matters for ChatGPT** — Articles >2,900 words avg 5.1 ChatGPT citations vs 3.2 for sub-800-word pieces
10. **Internal linking** — Link related topic pages to each other. Helps AI understand entity relationships and topical coverage within your site

### Optimization Priority Order

When deciding where to invest effort, follow this order (meta tags are polish, not the engine):

1. High-quality main content
2. Clear headings & section structure (H1→H2→H3)
3. FAQs & direct answers
4. Internal linking between related topics
5. Schema markup
6. Meta titles & descriptions
7. ~~Meta tags alone~~ — lowest impact on LLM visibility

### Chunk Quality Checklist
- [ ] Does this section independently answer a specific question?
- [ ] Is the key information in the first 2-3 sentences?
- [ ] Are claims backed by verifiable data with attribution?
- [ ] Is the language entity-based (concepts + relationships) vs keyword-stuffed?
- [ ] Could an AI extract and quote this without additional context?

## Technical Audit Workflow

When auditing a site for AI search readiness:

1. **Crawler access** — Check robots.txt allows: GPTBot, ClaudeBot, PerplexityBot, Google-Extended, Googlebot
2. **llms.txt** — Check if `yourdomain.com/llms.txt` exists and is properly structured
3. **Schema markup** — Verify FAQPage, Article, Organization, Person schema on key pages
4. **HTML rendering** — Check if content requires JavaScript (many AI crawlers can't execute JS)
5. **Page speed** — FCP < 0.4s averages 6.7 AI citations; FCP > 1.13s drops to 2.1 citations. Fast pages are 3x more likely to be cited
6. **Heading hierarchy** — Verify proper H1→H2→H3 semantic structure
7. **Freshness dates** — Check for "Last updated" timestamps on content pages
8. **Alt text** — Verify images have descriptive alt text for multi-modal indexing
9. **Brand consistency** — Cross-check brand description across About page, social profiles, directories
10. **Intro strength** — First 30% of content generates 44.2% of citations; audit whether key answers appear early

Output a prioritized findings table: Issue | Severity | Impact on AI Visibility | Fix

## llms.txt Generation

Create at `yourdomain.com/llms.txt` in Markdown format:

```markdown
# [Brand/Site Name]

> [1-2 sentence description of the site and what it offers]

## [Category 1, e.g., "Blog" or "Guides"]

- [Page Title](URL): Brief description of content
- [Page Title](URL): Brief description of content

## [Category 2, e.g., "Products" or "Services"]

- [Page Title](URL): Brief description of content

## FAQ

- [FAQ Page](URL): Common questions about [topic]
```

Rules: H1 = site name (mandatory), blockquote = site summary, H2s = content categories, links = Markdown format with optional descriptions. Keep descriptions under 15 words. Prioritize pages with highest expertise signals.

## Schema Generation

Generate JSON-LD schema markup. Priority types for AI visibility:

| Schema | When to Use | AI Impact |
|--------|------------|-----------|
| FAQPage | Pages with Q&A content | Directly quotable by AI; highest extraction rate |
| Article | Blog posts, guides, editorials | Reinforces authority + freshness signals |
| Organization | Homepage, About page | Anchors brand identity for consistent AI recognition |
| Person | Author pages, team bios | Strengthens E-E-A-T; expert content gets more trust |
| HowTo | Tutorial/instruction pages | Highly extractable step format |
| Product | Product pages | Supports AI product recommendations |
| VideoObject | Pages with video | Multi-modal citation support |
| Dataset | Market data, price tables, statistics pages | Signals machine-readable data; important for finance/crypto sites |

When generating schema, always output valid JSON-LD with `<script type="application/ld+json">` wrapper. Include: author, datePublished, dateModified, publisher, and description fields.

## AI-Optimized Content Writing

When writing new content designed for AI citation:

1. **Structure:** H1 title → TL;DR paragraph → H2 sections (one topic each) → FAQ section at end
2. **Each section:** Answer-first sentence → supporting evidence → verifiable data point
3. **Original value:** Include proprietary data, unique analysis, or expert perspective — AI already knows generic information and won't search for it
4. **Quotability:** Write sentences that can stand alone when extracted. Avoid pronouns referring to other sections
5. **Attribution:** Cite sources inline ("According to [Source], ..."). AI systems trust content that cites its own sources
6. **Meta signals:** Include author byline with credentials, publication date, last-updated date

## Platform-Specific Differences

For detailed platform optimization (ChatGPT vs Perplexity vs Google AI vs Gemini), see [references/platform-playbooks.md](references/platform-playbooks.md).

Quick reference:
- **Google AI Overviews:** Traditional SEO + schema + multi-modal content. 76% of citations from Google top 10. 47% of citations from pages below position #5
- **Google AI Mode:** Separate system — only 13.7% citation overlap with AI Overviews. 93% zero-click rate (vs 43% for AI Overviews). Optimize separately
- **ChatGPT:** Training data + Wikipedia (47.9% of top citations) + Bing SEO. Direct sources beat aggregators by +11 pts
- **Perplexity:** Content freshness (refresh every 2-3 days) + brand mentions (~35% of inclusions) + Reddit/forums (Reddit = 46.7% of citations)
- **Gemini:** Google ecosystem (Business Profile, YouTube, structured data). Now 18.2% chatbot market share

## Key Statistics (2026)

### Traffic & Conversion
- AI search visitors convert at 15.9% (ChatGPT referral) vs 1.76% organic — ~9x better
- AI-referred sessions surged 527% year-over-year
- 93% of AI Mode searches end without a click vs 43% for AI Overviews
- Brands receive citations through third-party sources 6.5x more than their own domains

### Citation Patterns
- 44.2% of all LLM citations come from the first 30% of text (intro section)
- Pages with schema markup are 36% more likely to appear in AI summaries
- Only 11% of domains are cited by both ChatGPT and Google AI Overviews
- Only 13.7% citation overlap between Google AI Mode and Google AI Overviews
- Wikipedia = 47.9% of ChatGPT's top-10 citations; Reddit = 46.7% of Perplexity's citations
- 47% of AI Overview citations come from pages ranking below position #5 (DA correlation dropped to r=0.18)

### Content Factors
- Articles >2,900 words avg 5.1 ChatGPT citations vs 3.2 for sub-800-word pieces
- Content updated within 3 months gets ~6 citations vs 3.6 for older content
- Pages with FCP < 0.4s avg 6.7 citations; FCP > 1.13s drops to 2.1
- Multi-modal content (text+image+video+data) has r=0.92 correlation with AI citation
- Content freshness drops Perplexity visibility within 2-3 days without updates

### Market
- GEO market valued at $848M (2025), projected $33.7B by 2034 (50.5% CAGR)
- ChatGPT: 68% chatbot market share (down from 87.2%); Gemini: 18.2% (up from 5.4%)
