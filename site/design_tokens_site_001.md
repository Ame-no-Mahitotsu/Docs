---
ID: design_tokens_site_001
Type: session
Role: ux
Topic: Tailwind Design Token Spec — Direction C (Festa d'Inverno) — APPROVED
Date: 2026-03-23
Participants: Sarah (UX), Alessandro (Tech Lead review)
Ticket: ISS-100
Sprint: site-1
Status: closed
Project: site
Outputs: tailwind.config.ts token spec, next/font/google integration spec, component utility class guide, accessibility notes
---

# Tailwind Design Token Spec — Direction C: Festa d'Inverno
**Ticket:** ISS-100
**Status:** Approved — G5 (Alessandro) changes applied 2026-03-23
**For:** Ash (Frontend) — implement directly into `tailwind.config.ts` and `app/layout.tsx`

---

## Font Loading — `app/layout.tsx`

**Important:** Do NOT use a `<link>` tag to Google Fonts. Use `next/font/google` — required for App Router font optimisation and self-hosting.

```ts
// app/layout.tsx
import { Libre_Baskerville, Lato } from 'next/font/google'

const libreBaskerville = Libre_Baskerville({
  subsets: ['latin'],
  weight: ['400', '700'],
  style: ['normal', 'italic'],
  variable: '--font-serif',
})

const lato = Lato({
  subsets: ['latin'],
  weight: ['300', '400', '700'],
  variable: '--font-sans',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="it" className={`${libreBaskerville.variable} ${lato.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

---

## `tailwind.config.ts` — Full Token Spec

All tokens go inside `theme.extend` — this preserves all Tailwind defaults alongside custom tokens.

```ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './app/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './lib/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {

      // ─── COLOURS ────────────────────────────────────────────────────────────
      colors: {
        // Surfaces
        'surface':       '#F5F0E8',  // main page background — warm cream
        'surface-dark':  '#1B3A5C',  // midnight blue — nav, hero, footer, CTA band
        'surface-card':  '#EDE6D6',  // product cards, secondary panels
        'surface-glow':  '#FDECC8',  // candlelight — quote strip, highlight blocks
        'surface-deep':  '#122540',  // hero image bg — darker than nav

        // Text
        'ink':           '#221C16',  // primary text on light surfaces
        'ink-muted':     '#6B5E4E',  // secondary text, captions, labels
        'ink-light':     '#BFD0E3',  // text on dark surfaces (nav, hero, footer)
        'ink-dim':       '#7A8FA0',  // secondary text on dark — placeholders, muted

        // Accents
        'gold':          '#C9943A',  // primary interactive accent — CTAs on dark, eyebrow, hover borders
        'gold-hover':    '#B8832A',  // gold at hover
        'sienna':        '#A0522D',  // warm accent — primary CTAs on light surfaces, active links
        'sienna-dark':   '#7A3D1E',  // sienna at hover

        // Borders
        'border':        '#D4C5A9',  // borders on light surfaces
        // NOTE: border-dark uses a static rgba — cannot use Tailwind opacity modifiers (by design)
        'border-dark':   'rgba(191, 208, 227, 0.15)', // borders on dark surfaces
      },

      // ─── FONT FAMILIES ──────────────────────────────────────────────────────
      fontFamily: {
        serif: ['var(--font-serif)', 'Georgia', 'serif'],
        sans:  ['var(--font-sans)',  'system-ui', 'sans-serif'],
      },

      // ─── FONT SIZES ─────────────────────────────────────────────────────────
      // Use these tokens — do NOT use default Tailwind size scale (text-4xl etc.)
      fontSize: {
        // Display — hero and section titles
        'display-xl': ['3.6rem',   { lineHeight: '1.15', letterSpacing: '-0.01em' }],
        'display-lg': ['2.8rem',   { lineHeight: '1.2',  letterSpacing: '-0.005em' }],
        'display-md': ['2rem',     { lineHeight: '1.3',  letterSpacing: '0' }],

        // Heading — card titles, subsections
        'heading-lg': ['1.3rem',   { lineHeight: '1.4',  letterSpacing: '0' }],
        'heading-md': ['1.1rem',   { lineHeight: '1.45', letterSpacing: '0' }],

        // Body
        'body-lg':    ['1rem',     { lineHeight: '1.75', letterSpacing: '0' }],
        'body-md':    ['0.95rem',  { lineHeight: '1.75', letterSpacing: '0' }],
        'body-sm':    ['0.875rem', { lineHeight: '1.7',  letterSpacing: '0' }],

        // Labels / eyebrows — always paired with `uppercase`
        'label-lg':   ['0.8rem',   { lineHeight: '1.4',  letterSpacing: '0.14em' }],
        'label-md':   ['0.75rem',  { lineHeight: '1.4',  letterSpacing: '0.16em' }],
        'label-sm':   ['0.7rem',   { lineHeight: '1.4',  letterSpacing: '0.2em' }],

        // Pull quotes
        'quote-lg':   ['2rem',     { lineHeight: '1.5',  letterSpacing: '0' }],
        'quote-md':   ['1.5rem',   { lineHeight: '1.5',  letterSpacing: '0' }],
      },

      // ─── SPACING ────────────────────────────────────────────────────────────
      spacing: {
        '18':         '4.5rem',
        '22':         '5.5rem',
        '26':         '6.5rem',
        '30':         '7.5rem',
        'section-y':  '5rem',    // standard vertical section padding — use as py-section-y
        'page-x':     '4rem',    // horizontal page padding desktop — use as px-page-x
        'page-x-sm':  '1.5rem',  // horizontal page padding mobile — use as px-page-x-sm
      },

      // ─── SHADOWS ────────────────────────────────────────────────────────────
      boxShadow: {
        'card':       '0 2px 12px rgba(34, 28, 22, 0.07)',
        'card-hover': '0 8px 24px rgba(34, 28, 22, 0.12)',
        'glow-gold':  '0 0 40px rgba(201, 148, 58, 0.2)',   // candlelight on hero image
        'glow-sm':    '0 0 20px rgba(201, 148, 58, 0.12)',  // selected product card
      },

    },
  },
  plugins: [],
}

export default config
```

---

## Component Utility Classes — Reference for Ash

### Navbar (dark)
```
bg-surface-dark text-ink-light
Logo:         font-serif font-normal text-surface-glow
Nav link:     font-sans font-normal text-label-md uppercase text-ink-light opacity-80 hover:opacity-100 hover:text-gold transition-opacity
Lang active:  text-gold font-bold
Lang inactive:text-ink-light opacity-60
```

### Hero (dark full-bleed)
```
Wrapper:      bg-surface-dark overflow-hidden relative
Eyebrow:      font-sans font-bold text-label-md uppercase text-gold
Title:        font-serif font-bold text-display-xl text-surface-glow  [key word in italic]
Subtitle:     font-sans font-light text-body-lg text-ink-light opacity-85
CTA primary:  bg-gold text-surface-dark font-sans font-bold text-label-lg uppercase px-8 py-3 hover:bg-gold-hover transition-colors
CTA secondary:text-ink-light border-b border-border-dark text-label-md uppercase opacity-70
```

### Festive banner (gold strip between hero and quote)
```
Wrapper:      bg-gold
Text:         font-sans font-bold text-label-sm uppercase text-surface-dark
Separators:   text-surface-dark opacity-40
```

### Quote strip (candlelight)
```
Wrapper:      bg-surface-glow border-y-[3px] border-gold py-18 px-page-x text-center
Quote:        font-serif italic text-quote-lg text-ink
Cite:         font-sans font-bold text-label-md uppercase text-sienna
```

### Product card
```
Wrapper:      bg-surface-card border-b-[3px] border-transparent hover:border-gold
              shadow-card hover:shadow-card-hover transition-all hover:-translate-y-1
Name:         font-serif font-bold text-heading-md text-ink
Size:         font-sans font-light text-body-sm text-ink-muted
CTA:          bg-sienna text-white font-sans font-bold text-label-sm uppercase px-4 py-2
              hover:bg-sienna-dark transition-colors
```

### CTA band (dark, with Gabriele quote)
```
Wrapper:      bg-surface-dark py-section-y px-page-x text-center
Title:        font-serif italic font-normal text-display-md text-surface-glow
Body:         font-sans font-light text-body-md text-ink-light opacity-80
CTA:          same as hero CTA primary
```

### Footer (dark)
```
Wrapper:      bg-surface-dark border-t border-border-dark py-8 px-page-x
Text:         font-sans font-light text-label-sm text-ink-light opacity-50
Link hover:   hover:text-gold transition-colors
```

---

## Accessibility — Contrast Reference

| Token pair | Ratio | WCAG | Use |
|---|---|---|---|
| `gold` on `surface-dark` | ~5.8:1 | ✅ AA | eyebrow, CTA on dark, hover states |
| `surface-glow` on `surface-dark` | ~13.2:1 | ✅ AAA | hero title, logo |
| `ink-light` on `surface-dark` | ~6.1:1 | ✅ AA | body text on dark |
| `ink` on `surface-glow` | ~13.2:1 | ✅ AAA | quote text |
| `sienna` on `surface` | ~4.6:1 | ✅ AA large | CTAs on light (18px+ only) |
| `ink-muted` on `surface-card` | ~4.2:1 | ✅ AA large | size labels only — not body |
| `gold` on `surface` | ~2.9:1 | ❌ FAIL | **decorative only — never as text on light bg** |

**Hard rule for Ash:** `gold` is never used as a text colour on `surface`, `surface-card`, or `surface-glow`. Decorative borders and backgrounds only on light surfaces.
