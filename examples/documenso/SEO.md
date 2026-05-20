---
devmd: seo
version: "1.0"
project: documenso
updated: 2026-05-13
status: minimal

type: saas-application
note: "Documenso is primarily a SaaS app behind authentication. SEO applies to public-facing pages only."

public_pages:
  - "/signin"
  - "/signup"
  - "/d/{directLinkToken}"  # Direct link signing
  - Marketing site (separate)

seo_relevant: false  # Most app pages are behind auth
geo_relevant: false  # No location-based content optimization
---

# SEO.md — Documenso

## Status

Documenso is a **SaaS application** where the vast majority of pages are behind authentication. Traditional SEO optimization applies only to a small set of public-facing pages.

## Meta Tags

### Default Meta

```html
<meta name="robots" content="noindex, nofollow" />
```

Most app routes use `noindex` because they contain user-specific data.

### Public Pages

```html
<!-- /signin, /signup -->
<title>Sign In | Documenso — Open Source Document Signing</title>
<meta name="description" content="Sign documents for free with Documenso, the open-source alternative to DocuSign." />
<meta name="robots" content="index, follow" />

<!-- Open Graph -->
<meta property="og:title" content="Documenso — The Open Source DocuSign Alternative" />
<meta property="og:description" content="Document signing should be open, beautiful, and accessible to everyone." />
<meta property="og:image" content="https://documenso.com/og-image.png" />
<meta property="og:type" content="website" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@documenso" />
```

## CSP and Embedding

The most SEO-relevant security configuration is `frame-ancestors`, which controls where Documenso can be embedded:

```
Content-Security-Policy: frame-ancestors 'self' {configured-origins};
```

- Default: `'self'` only (no cross-origin embedding)
- Enterprise embedding: Configurable list of allowed origins
- This affects how search engines perceive the embeddable signing experience

See @SECURITY.md#content-security-policy for full CSP configuration.

## Direct Link Pages

Direct links (`/d/{token}`) are the only dynamically-generated public content:

- These are **not indexed** (`noindex`) because they contain document-specific data
- They should load fast for recipient experience (not SEO)
- Server-side rendered for immediate content display

## Performance Considerations

While not SEO-critical for authenticated pages, performance matters for direct link signing:

- **SSR**: React Router v7 server-side rendering ensures fast first paint
- **Code splitting**: Per-route code splitting via Vite
- **Image optimization**: Signature images are base64 encoded (no additional requests)
- **PDF loading**: Streamed from storage provider, progressive page rendering

## Structured Data

No JSON-LD structured data is used in the application. The marketing site (separate from this codebase) handles structured data for search visibility.

## Sitemap

No sitemap is generated from the application. All indexable content lives on the separate marketing site.

## Cross-References

- CSP configuration: @SECURITY.md#content-security-policy
- Direct link feature: @GLOSSARY.md#direct-link
- Public routes: @UI.md#unauthenticated-routes
- Brand messaging for meta: @BRAND.md
