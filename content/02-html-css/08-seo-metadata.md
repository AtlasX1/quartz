# SEO & Metadata: L4 Engineering Guide

## Part 1: Meta Tags & SEO Fundamentals

### 1.1 Critical Meta Tags for Search Engine Optimization

Meta tags communicate page information to search engines before they render content. They don't affect on-page display but directly influence how pages are indexed and presented in search results.

**Character encoding** must appear in the first 1024 bytes of the HTML document. Modern UTF-8 encoding is standard; legacy encodings cause parsing errors and encoding-related rendering issues.

**Viewport meta tag** signals responsive design adoption. Without it, mobile browsers apply virtual viewports larger than device width, breaking responsive layouts. `width=device-width` matches CSS pixels to device pixels; `initial-scale=1.0` sets initial zoom.

**Title tags** appear in search results (SERP snippets) and browser tabs. Search engines truncate titles exceeding ~60 characters on desktop, ~50 on mobile. Title should contain primary keyword (but not keyword-stuffed); front-load important words since users typically read left to right.

**Meta descriptions** are optional but recommended. Google uses them to generate search result snippets, though it reserves the right to generate its own description if it deems yours inadequate. Length should be 150-160 characters—longer descriptions are truncated; shorter ones waste space.

**Canonical links** prevent duplicate content penalties. When identical or near-identical content exists at multiple URLs (due to URL parameters, printer-friendly versions, or content distribution), the canonical link designates the authoritative version for indexing.

### 1.2 Crawling & Indexing Control

The `robots` meta tag controls whether pages are indexed and whether links are followed. `index` (default) allows indexing; `noindex` prevents it. `follow` (default) allows link following; `nofollow` prevents it.

Robots directives represent a trust spectrum: `index, follow` means crawl and index everything; `noindex, follow` prevents indexing but allows link discovery (useful for staging/draft pages); `noindex, nofollow` completely isolates the page from search engine crawling.

The trade-off: `noindex` prevents duplicate content penalties but also prevents ranking. It's temporary—used for pagination, filters, or staging—and should eventually be removed.

---

## Part 2: Social Media & Open Graph

### 2.1 Open Graph Protocol Theory

Open Graph (OG) tags control how pages are presented when shared on social platforms. Without OG tags, social platforms attempt to extract title, image, and description from page content, which often yields suboptimal results (wrong image selection, truncated text, missing context).

OG tags are Facebook's protocol but are widely adopted by Twitter, LinkedIn, Slack, and others. They're simple metadata: `<meta property="og:title" content="...">` establishes a key-value pair that social platforms can parse and display.

**Strategic implications:**

- **og:title**: Often differs from page `<title>` tag because social shares have space constraints and different audience expectations
- **og:image**: Should be 1200×630px (Facebook's recommended size); platforms resize but quality degrades if original dimensions are significantly different
- **og:description**: Similar to but separate from meta description—optimized for social context, not SERP display
- **og:url**: Establishes canonical URL for sharing; prevents duplicate sharing records if content exists at multiple URLs

The fundamental purpose: control perception and engagement metrics. A well-crafted OG image (thumbnail) increases click-through rate on social shares by 30-100% compared to algorithmically selected images.

### 2.2 Twitter Cards

Twitter Cards function similarly to Open Graph but use different metadata structure (`twitter:card` instead of `og:type`). Twitter Cards support several card types: summary (small image), summary_large_image (large image), player (video/audio), app.

The practical distinction: OG metadata is universal, applicable to all platforms; Twitter-specific tags allow platform-specific optimization (e.g., card format, account attribution via `twitter:creator`).

---

## Part 3: Structured Data & Schema.org

### 3.1 JSON-LD as Structured Data Format

Structured data enables search engines to understand content semantically. Rather than inferring "this page is about a product," structured data explicitly states "this is a Product with name, price, and rating."

**JSON-LD (JavaScript Object Notation for Linked Data)** is the recommended format for structured data. It's embedded in `<script type="application/ld+json">` tags and doesn't affect page rendering. Alternatives like microdata (`itemscope`, `itemtype`) and RDFa exist but are less recommended.

**Schema.org types** are standardized schemas for common content types: Product, Article, Recipe, Event, Organization, LocalBusiness, etc. Each schema defines expected properties (e.g., Product schema expects name, price, availability).

**Search engine implications**: Structured data enables **rich snippets**—enhanced search result display showing prices, ratings, author, publication date. Rich snippets increase click-through rate (CTR) by 20-30% compared to plain-text results.

The relationship between content, schema types, and search results:

- **No schema**: Generic SERP display, no rich information
- **Basic schema**: Title, description, maybe an image
- **Comprehensive schema**: Star ratings, price, availability, author, publication date

### 3.2 Specific Schema Applications

**Article schema** marks blog posts, news articles, and long-form content. Includes headline, author, publication date, modified date, and article body. Search engines use this to determine freshness and credibility.

**Product schema** enables e-commerce rich snippets (price, rating, availability). The schema supports aggregated ratings (star count and review count), which dramatically improves SERP appearance and CTR.

**Organization schema** establishes company information (name, logo, contact, location, social profiles). It's often placed site-wide, appearing in knowledge panels when users search the company name.

---

## Part 4: Sitemap & Technical SEO

### 4.1 XML Sitemaps & Crawl Efficiency

An XML sitemap is a file listing all URLs on a website with metadata (last modified date, update frequency, priority). It's not required—search engines can crawl links to discover pages—but sitemaps improve crawl efficiency, especially for large sites or sites with poor internal linking.

**Priority** values (0.0-1.0) hint to search engines which pages are most important, but search engines don't strictly honor priorities if they conflict with other signals (popularity, freshness). Effective sitemaps set realistic priorities: homepage 1.0, category pages 0.8, product pages 0.6, etc.

**changefreq** (never, yearly, monthly, weekly, daily) tells search engines how often to revisit pages. Static content marked "never" reduces crawl waste; dynamic content marked "weekly" encourages regular revisits.

Sitemaps should be registered in robots.txt (`Sitemap: https://example.com/sitemap.xml`) and submitted through Google Search Console and Bing Webmaster Tools.

### 4.2 Robots.txt & Crawl Directives

Robots.txt is a text file at the site root that controls search engine crawling behavior. It's processed by all major search engines (though it's advisory, not binding—malicious crawlers ignore it).

**Directives:**

- `User-agent: *` applies to all crawlers; specific agents can be targeted (e.g., `User-agent: Googlebot`)
- `Allow` permits crawling (usually explicit for paths that would otherwise be blocked)
- `Disallow` prevents crawling of specific paths
- `Crawl-delay` specifies seconds between requests (reduces server load)
- `Sitemap` links to the XML sitemap

**Strategic use**: Block admin pages, staging environments, search result pages, and duplicate content. Don't block important pages—they'll still be indexed if linked from elsewhere; robots.txt just prevents crawling, not indexing.

---

## Part 5: Core Web Vitals & Performance SEO

### 5.1 Web Vitals as Ranking Factors

Google incorporates Core Web Vitals (LCP, FID/INP, CLS) as ranking factors. Pages meeting thresholds rank higher than slower pages, all else equal. This creates a performance requirement for competitive SEO.

**LCP (Largest Contentful Paint) < 2.5s** measures visual completeness. Pages must render hero content (large images, headlines, primary content) quickly. Render-blocking resources (CSS, JavaScript) and unoptimized images are primary culprits.

**INP (Interaction to Next Paint) < 200ms** measures responsiveness to user input. Slow JavaScript execution during interactions (click handlers, scroll listeners) directly impacts rankings.

**CLS (Cumulative Layout Shift) < 0.1** measures visual stability. Fonts loading late, ads inserting, or dynamic content appearing causes layout shifts. These are disproportionately weighted by Google because they cause misclicks and poor user experience.

### 5.2 Mobile-First Indexing Implications

Google indexes and ranks mobile versions first, even for users searching on desktop. This fundamental shift means mobile optimization is no longer "nice-to-have"—it's the primary ranking version.

**Implications:**

- Mobile viewport must be at least 50-60 characters wide (enough for readable text)
- Touch targets must be 44×44px minimum
- Core Web Vitals are measured on mobile (typically faster desktop devices artificially inflate scores)
- Lazy loading and deferring below-fold content is essential on mobile

---

## Part 6: Semantic HTML & Crawlability

### 6.1 Heading Hierarchy & Content Structure

Heading hierarchy (`<h1>` through `<h6>`) signals content structure to search engines. A proper hierarchy indicates topical relationships and content importance.

**Rules:**

- **One H1 per page**: The primary page topic. Google's algorithm weighs H1 heavily for relevance
- **Sequential nesting**: H2s under H1, H3s under H2. Skipping levels (H1 directly to H3) signals poor structure
- **Keyword inclusion**: H1 should contain the primary keyword (naturally, not forced); H2s can target secondary keywords

The relationship between heading structure and search ranking: clear hierarchy improves topical relevance signals, potentially boosting rankings 10-20 positions for competitive keywords.

### 6.2 Semantic Elements & Search Understanding

Semantic elements (`<article>`, `<nav>`, `<main>`, `<aside>`, `<section>`) help search engines understand page structure and content type. They don't directly rank pages but improve context extraction.

---

## Part 7: SEO Audit Checklist

### Technical Requirements

- HTTPS enabled (ranking factor)
- Mobile-responsive design (detected via viewport meta tag)
- Core Web Vitals within thresholds (LCP < 2.5s, INP < 200ms, CLS < 0.1)
- Robots.txt present and optimized (disallow only necessary)
- Sitemap.xml submitted and indexed
- Canonical links for duplicate content

### Content Requirements

- Unique, high-quality content (avoid thin content < 300 words)
- Primary keyword in H1 (naturally)
- Proper heading hierarchy (H1 > H2 > H3)
- Alt text on images (descriptive, keyword-appropriate but not stuffed)
- Internal linking structure (topic clustering, pillar-to-cluster links)
- Meta descriptions present (unique per page, 150-160 chars)
- Descriptive URLs (avoid /page1, /page2; use /blog/keyword-topic)

### On-Page Signals

- Page title (50-60 chars, keyword-forward)
- Keyword appears in first paragraph
- Outbound links to authoritative sources (signals quality)
- Content length appropriate to query intent (competitive queries need 1500+ words; simple queries need less)

---

## Interview Questions

**Q1: A client's homepage isn't ranking despite good content. Audit checklist?**

Check: Core Web Vitals thresholds met, HTTPS enabled, Mobile-friendly, Canonical tag present, Meta description unique, H1 contains primary keyword, robots.txt allows indexing, Sitemap submitted, Internal linking structure exists. Most common issue: slow LCP (render-blocking resources) or non-existent schema markup.

**Q2: How do Open Graph tags improve SEO?**

They don't directly improve rankings but increase social sharing click-through rate (CTR) by 30-100% through better thumbnail/title display. Higher CTR signals popularity to search engines, which correlates with improved rankings. Indirect but measurable impact.

**Q3: Should you use noindex on pagination pages?**

No—use rel="next"/rel="prev" canonical links or infinite scroll instead. Noindex prevents indexing but also prevents ranking. Canonical consolidation is better: 10 page-1, page-2, page-3 URLs point canonical to page-1, consolidating ranking signals.

**Q4: Core Web Vitals impact rankings. Optimization priority?**

Focus on LCP first (biggest ranking impact). Eliminate render-blocking resources, preconnect to critical domains, lazy-load below-fold content. Then INP (JavaScript optimization). Then CLS (reserve layout space). Typical improvements: LCP 5s→2s, INP 500ms→150ms, CLS 0.5→0.05.

---

## Key Takeaways

1. **Meta tags control discovery and display**, not ranking directly
2. **Canonical links consolidate duplicate content** and prevent indexing fragmentation
3. **Open Graph improves social CTR** by 30-100%, indirectly boosting rankings
4. **JSON-LD structured data enables rich snippets**, increasing CTR 20-30%
5. **Core Web Vitals are direct ranking factors** - LCP/INP/CLS thresholds are non-negotiable
6. **Mobile-first indexing shifts SEO to mobile optimization** - mobile is the primary version
7. **Heading hierarchy signals content structure** - one H1 per page, sequential nesting

