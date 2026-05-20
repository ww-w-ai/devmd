---
devmd: seo
version: 0.1.0

rendering:
  strategy: ""                   # ssr | ssg | isr | csr
  html_first: true               # full content in initial HTML

meta:
  title_template: ""             # e.g. "{page} | {site}"
  description_max_chars: 160
  canonical_strategy: ""         # self | primary | none

og_image:
  generation: ""                 # satori | puppeteer | static
  dimensions: "1200x630"

structured_data:
  types: []                      # Article | Product | FAQ | Organization | ...
  article_body_included: true    # full text in JSON-LD for LLM citation

sitemap:
  path: "/sitemap.xml"
  news_sitemap: false            # /sitemap-news.xml
  changefreq: ""
  priority_rules: []

robots:
  ai_crawlers:
    ai_train: ""                 # yes | no
    ai_search: ""                # yes | no
    ai_input: ""                 # yes | no
  content_signal_header: false   # Content-Signal HTTP header

geo:
  markdown_endpoint: false       # /:slug.md for AI agents
  speakable: false               # speakable JSON-LD for voice
  hreflang: []                   # e.g. [en, ko, ja]

cache:
  strategy: ""                   # immutable | stale-while-revalidate | ...
  ttl: ""
  purge_on: []                   # e.g. [create, update, delete]

performance:
  lcp_target_ms: 0
  cls_target: 0
  fid_target_ms: 0
---

# SEO.md

> SEO + GEO strategy — rendering, meta, structured data, AI crawler policy, and performance.

## Rendering Strategy

<!-- SSR/SSG/ISR choice. Reference @ARCHITECTURE.md#tech-stack. -->

## Meta Tags

<!-- Title template, description rules. Reference @BRAND.md#copy-rules. -->

## Structured Data

<!-- JSON-LD types. Reference @SCHEMA.md for data models. -->

## Sitemap & Robots

<!-- Sitemap config, AI crawler rules. -->

## GEO (Generative Engine Optimization)

<!-- Markdown endpoints, speakable, hreflang. Reference @API.md if markdown API exists. -->

## Cache Strategy

<!-- CDN caching, purge triggers. Reference @INFRA.md#cdn. -->

## Performance Targets

<!-- Core Web Vitals. Reference @OPERATIONS.md#slos. -->
