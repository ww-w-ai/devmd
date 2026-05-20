# SEO.md Specification

> Version: 0.1.0-draft | Status: Draft

## Purpose

SEO.md defines SEO (Search Engine Optimization) and GEO (Generative Engine Optimization) strategy as a declarative specification. It is the first standard to treat SEO configuration as a machine-readable spec file rather than scattered implementation details. Derived from production-tested patterns across multiple projects.

## Frontmatter Schema

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `name` | `string` | REQUIRED | Site or product name |
| `canonical` | `enum(auto\|manual)` | OPTIONAL | Canonical URL strategy. Default: `auto`. |
| `rendering` | `enum(ssr-html-first\|ssr\|csr\|ssg\|isr)` | REQUIRED | Primary rendering strategy |
| `meta` | `Meta` | REQUIRED | Default meta tag configuration |
| `og_image` | `OGImage` | OPTIONAL | Open Graph image generation |
| `structured_data` | `StructuredData` | OPTIONAL | JSON-LD configuration |
| `sitemap` | `Sitemap` | OPTIONAL | Sitemap generation rules |
| `robots` | `Robots` | OPTIONAL | Robots and AI crawler policies |
| `geo` | `GEO` | OPTIONAL | Generative Engine Optimization settings |
| `cache` | `map<string, string>` | OPTIONAL | Content type to cache policy map (e.g., `"static": "immutable, max-age=31536000"`) |
| `performance` | `Performance` | OPTIONAL | Core Web Vitals targets |

### Meta

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `title_template` | `string` | REQUIRED | Title template with `{page}` placeholder (e.g., `"{page} | MySite"`) |
| `default_description` | `string` | REQUIRED | Fallback meta description. MUST NOT exceed 160 characters. |
| `robots` | `string` | OPTIONAL | Default robots meta value (e.g., `"index, follow"`) |
| `twitter_card` | `enum(summary\|summary_large_image)` | OPTIONAL | Twitter card type |

### OGImage

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `generator` | `string` | REQUIRED | Generator tool (e.g., `"satori"`, `"puppeteer"`, `"cloudinary"`) |
| `size` | `number[2]` | REQUIRED | Width and height in pixels (e.g., `[1200, 630]`) |
| `font` | `string` | OPTIONAL | Font used in OG images |
| `cache` | `string` | OPTIONAL | Cache policy for generated images |

### StructuredData

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `site` | `string` | OPTIONAL | Site-level JSON-LD type (e.g., `"WebSite"`, `"Organization"`) |
| `pages` | `map<string, string>` | OPTIONAL | Page path to JSON-LD type map (e.g., `"/blog/*": "Article"`) |
| `breadcrumb` | `boolean` | OPTIONAL | Enable BreadcrumbList JSON-LD. Default: `false`. |

### Sitemap

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `dynamic` | `boolean` | REQUIRED | Whether sitemap is dynamically generated |
| `max_urls` | `number` | OPTIONAL | Maximum URLs per sitemap file. Default: `50000`. |
| `news_sitemap` | `boolean` | OPTIONAL | Generate separate `/sitemap-news.xml`. Default: `false`. |

### Robots

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `allow` | `string[]` | OPTIONAL | Allowed paths |
| `disallow` | `string[]` | OPTIONAL | Disallowed paths |
| `ai_crawlers` | `AICrawlers` | OPTIONAL | AI-specific crawler policies |

### AICrawlers

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `allow` | `string[]` | OPTIONAL | AI crawlers explicitly allowed (e.g., `["GPTBot", "ClaudeBot", "Google-Extended"]`) |
| `block_training` | `boolean` | OPTIONAL | Block AI training while allowing search. Default: `false`. |

### GEO

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `markdown_endpoint` | `boolean` | OPTIONAL | Expose `/:slug.md` endpoints for AI agents. Default: `false`. |
| `llms_txt` | `boolean` | OPTIONAL | Serve `/llms.txt` file. Default: `false`. |
| `html_first` | `boolean` | OPTIONAL | SSR full content in HTML without JS dependency. Default: `false`. |
| `citation_friendly` | `boolean` | OPTIONAL | Include `articleBody` in JSON-LD for LLM citation. Default: `false`. |

### Performance

| Field | Type | Presence | Description |
|-------|------|----------|-------------|
| `lcp_target` | `string` | OPTIONAL | Largest Contentful Paint target (e.g., `"2.5s"`) |
| `cls_target` | `string` | OPTIONAL | Cumulative Layout Shift target (e.g., `"0.1"`) |

## Body Sections

Sections MUST appear in this order when present.

| Section | Presence | Content Rules |
|---------|----------|---------------|
| `## Meta Tags` | REQUIRED | Meta tag strategy, per-page overrides, dynamic generation rules. |
| `## Structured Data` | OPTIONAL | JSON-LD types, schema.org usage, speakable markup. |
| `## Sitemap` | OPTIONAL | Sitemap generation strategy and update frequency. |
| `## Robots & AI Crawlers` | OPTIONAL | Crawler policies, Content-Signal headers, training opt-out. |
| `## GEO Strategy` | OPTIONAL | Generative Engine Optimization approach and endpoints. |
| `## Performance` | OPTIONAL | Core Web Vitals targets and optimization strategy. |
| `## Constraints` | OPTIONAL | SEO constraints or known limitations. |

## Cross-References

- SHOULD reference `@PRODUCT.md#tagline` for title and description source material.
- SHOULD reference `@BRAND.md` for tone alignment in meta descriptions.
- SHOULD reference `@INFRA.md` for CDN and cache layer configuration.

## Validation Rules

1. `meta.default_description` MUST NOT exceed 160 characters.
2. `meta.title_template` MUST contain a `{page}` placeholder.
3. `og_image.size` MUST be an array of exactly 2 numbers.
4. `sitemap.max_urls` MUST NOT exceed 50000 (sitemap protocol limit).
5. `rendering` MUST be one of the defined enum values.

## Conflict Detection

- `rendering` value SHOULD be consistent with `@INFRA.md` deployment target (e.g., `csr` with a static host, `ssr` with a server runtime).
- `cache` policies SHOULD NOT conflict with `@INFRA.md#cdn` cache configuration.
- `robots.disallow` paths SHOULD NOT include pages that require indexing per `@PRODUCT.md` goals.
