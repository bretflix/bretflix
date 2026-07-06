# Bretflix Alternate Theme — Handoff Plan

> **How to use this file:** Start a Claude Code session with the **`bretmcg/bretmcg.com`** repository attached, paste this file (or its contents) in, and say "implement this plan." Everything below is self-contained.

## Goal

One codebase, two branded deployments on Vercel:

| Skin | Domain | Feel |
|---|---|---|
| **bretmcg** (skin 1) | bretmcg.com | The site exactly as it is today — zero visible changes |
| **bretflix** (skin 2) | bretflix.com | Netflix-parody dark theme with heavy "Bretflix" wording |

The Bretflix skin should feel like a genuinely different site, not a recolor: different name, tagline, section headings, button copy, footer, metadata, logo, and favicon.

## Architecture: one repo, two Vercel projects, one env var

Create a **second Vercel project** that imports the same `bretmcg/bretmcg.com` repo. A build-time env var selects the skin:

- Existing Vercel project → domain `bretmcg.com` → `NEXT_PUBLIC_SITE_THEME` unset (or `bretmcg`) → default skin.
- New Vercel project (name it `bretflix`) → domain `bretflix.com` → `NEXT_PUBLIC_SITE_THEME=bretflix`.

Because `NEXT_PUBLIC_*` vars are inlined at **build time**, each domain gets its own fully prerendered build — no runtime host detection, no middleware, no flash of the wrong theme, clean per-domain SEO. Every push to the production branch deploys both projects automatically.

*(Rejected alternative: one Vercel project serving both domains with host-based middleware theming. It saves a project but forces per-request logic, breaks static prerendering of brand copy, and complicates canonicals/OG. Two projects is the standard Vercel pattern.)*

## Implementation steps

> The site is a Next.js app. **Explore the repo first** and adapt paths: find the root layout (`app/layout.tsx`, or `pages/_app.tsx` + `pages/_document.tsx`), the global stylesheet, whether Tailwind is used, and every component containing user-facing copy (header/nav, hero, section headings, cards, footer, 404).

### 1. Brand config module — single source of truth (`lib/brand.ts`)

```ts
export type BrandKey = 'bretmcg' | 'bretflix';

const key: BrandKey =
  process.env.NEXT_PUBLIC_SITE_THEME === 'bretflix' ? 'bretflix' : 'bretmcg';

const brands = {
  bretmcg: {
    key: 'bretmcg' as const,
    siteName: 'Bret McGowen',          // ← copy the site's existing title verbatim
    domain: 'https://bretmcg.com',
    tagline: '…existing tagline…',      // ← copy existing copy verbatim
    logoSrc: '…existing logo path…',
    faviconSrc: '…existing favicon…',
    ogImage: '…existing OG image…',
    nav: { /* existing nav labels */ },
    sections: {
      talks: 'Talks',
      videos: 'Videos',
      projects: 'Projects',
      about: 'About Bret McGowen',
    },                                   // ← match the site's real section names
    cta: { primary: '…existing…', secondary: '…existing…' },
    footer: '…existing footer text…',
    notFound: '…existing 404 copy (if any)…',
  },
  bretflix: {
    key: 'bretflix' as const,
    siteName: 'BRETFLIX',
    domain: 'https://bretflix.com',
    tagline: 'Unlimited Bret. No subscription required.',
    logoSrc: '/bretflix/wordmark.svg',
    faviconSrc: '/bretflix/favicon.svg',
    ogImage: '/bretflix/og.png',
    nav: { /* Netflix-flavored equivalents */ },
    sections: {
      talks: 'Trending Now',
      videos: 'Continue Watching',
      projects: 'Top Picks for You',
      about: 'Because You Watched: Bret McGowen',
    },
    cta: { primary: '▶ Play', secondary: '+ My List' },
    footer: 'Bretflix — binge responsibly. Questions? Contact Bret.',
    notFound: 'This title is not available in your region.',
  },
} satisfies Record<BrandKey, unknown>;

export const brand = brands[key];
```

Rules:
- **Every** brand-varying string, logo path, and metadata value flows through `brand`. Components never hardcode copy that differs between skins.
- The `bretmcg` entry must reproduce today's copy **verbatim** so bretmcg.com is byte-identical after the refactor.

### 2. Root layout wiring

- Set `<html data-theme={brand.key} …>` (App Router: in `app/layout.tsx`; Pages Router: `<Html>` in `_document.tsx` — env var is available at build time in both).
- Derive metadata from `brand`:
  - `metadataBase: new URL(brand.domain)` → correct canonical/OG URLs per domain.
  - Title template: `` `%s | ${brand.siteName}` ``, description from `brand.tagline`.
  - Favicon + OG image paths from `brand`.

### 3. CSS theming via variables

- Extract the current hardcoded colors in the global stylesheet into `:root` variables — these become the bretmcg defaults: `--bg`, `--fg`, `--accent`, `--surface`, `--muted`, etc.
- Add the Bretflix override block:

```css
[data-theme='bretflix'] {
  --bg: #141414;        /* Netflix near-black */
  --fg: #ffffff;
  --accent: #e50914;    /* Netflix red */
  --surface: #1f1f1f;   /* dark cards */
  --muted: #b3b3b3;
}
```

- If Tailwind: map tokens in the config (`colors: { accent: 'var(--accent)', … }`) so existing utility classes pick up the theme automatically. If Tailwind v4, use `@theme` / CSS variables directly.
- Style the Bretflix header as a Netflix-style top bar: wordmark left, dark translucent background; bold condensed uppercase headings.

### 4. Bretflix wording sweep (the "feels different" part)

Replace hardcoded strings with `brand.*` lookups everywhere, then apply this Bretflix copy deck:

- **Hero:** big red **BRETFLIX** wordmark + "Unlimited Bret. Talks, videos, and side projects — streaming now." Buttons: **▶ Play** (primary red) and **+ My List** (secondary gray).
- **Section headings:** Talks → **Trending Now** · Videos → **Continue Watching** · Projects → **Top Picks for You** · About → **Because You Watched: Bret McGowen**.
- **Cards:** small corner badges on recent items — `NEW EPISODE`, `TOP 10` — and hover text "Now streaming". (Bretflix-only: gate on `brand.key === 'bretflix'`.)
- **Footer:** "Bretflix — binge responsibly." + "Questions? Contact Bret."
- **404 page:** "This title is not available in your region."
- Sprinkle "Bretflix Original" as a label on Bret's own projects/talks.

### 5. Bretflix assets (`public/bretflix/`)

- `wordmark.svg` — pure SVG text: bold condensed uppercase "BRETFLIX" in `#E50914` on transparent (no external fonts; use `font-family="Arial Black, Impact, sans-serif"` with wide letter-spacing, or trace simple paths).
- `favicon.svg` — red "B" on `#141414`.
- `og.png` — 1200×630, dark background, red wordmark, tagline.

### 6. Vercel setup (manual dashboard steps — also add these to the repo README)

1. Vercel dashboard → **Add New… → Project** → import `bretmcg/bretmcg.com` a second time → name it **bretflix**.
2. New project → Settings → **Environment Variables** → `NEXT_PUBLIC_SITE_THEME = bretflix` for Production **and** Preview.
3. New project → Settings → **Domains** → add `bretflix.com` and `www.bretflix.com` (www → apex redirect); update the domain's DNS at the registrar per Vercel's instructions (A `76.76.21.21` / CNAME `cname.vercel-dns.com`, or Vercel nameservers).
4. (Optional) Existing project: explicitly set `NEXT_PUBLIC_SITE_THEME = bretmcg` — the code defaults to bretmcg anyway.
5. Redeploy both projects.

## Verification

1. `npm run dev` (no env var) → site is copy- and pixel-identical to current bretmcg.com.
2. `NEXT_PUBLIC_SITE_THEME=bretflix npm run dev` → dark/red skin, BRETFLIX wordmark, all reworded sections, `<title>` contains Bretflix, canonical points at bretflix.com.
3. Both `npm run build` variants succeed.
4. Screenshot both skins (Playwright/Chromium) and compare side by side.
5. After Vercel setup: confirm each production domain serves the right skin and correct canonical/OG URLs.

## Expected files touched

- `lib/brand.ts` *(new)*
- root layout (`app/layout.tsx` or `pages/_app.tsx` + `pages/_document.tsx`)
- global CSS (and `tailwind.config.*` if present)
- header/nav, hero, section, card, footer, 404 components (string swap → `brand.*`)
- `public/bretflix/wordmark.svg`, `favicon.svg`, `og.png` *(new)*
- `README.md` — dual-project Vercel setup notes
