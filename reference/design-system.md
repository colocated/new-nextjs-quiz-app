# Design system

A calm, content-first, utility-class design: **neutral surfaces + one violet accent**, generous
rounding, hairline borders, automatic dark mode. No component library, no custom CSS beyond the
tokens. Reproduce it exactly with these Tailwind classes.

## Principles

1. One accent: **violet-600** (primary action), violet-400 for hover borders/focus, violet-100/900 for tinted pills and selected states.
2. Surfaces are `white`/`neutral-50` in light and `neutral-950`/`neutral-900` in dark. Borders are *translucent* (`border-black/10`, `dark:border-white/10`) so they work on either surface.
3. Semantic colours only for meaning: emerald = correct/good, red = wrong/bad, amber = warning/mid/in-progress.
4. Rounded: cards `rounded-2xl`, options/inputs/buttons `rounded-xl` / `rounded-lg`, pills `rounded-full`.
5. Text hierarchy: `text-neutral-900/100` primary, `text-neutral-600 dark:text-neutral-400` secondary, `text-neutral-500` meta, `text-neutral-400` hints.
6. Motion is minimal: `transition` on hover, `transition-all` on the progress bar, `animate-spin` on the spinner. Nothing else.
7. Dark mode via **`dark:` variants driven by `prefers-color-scheme`** (Tailwind v4 default). No toggle.

## Tokens - `globals.css`

```css
@import "tailwindcss";

:root {
  --background: #ffffff;
  --foreground: #171717;
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --font-sans: var(--font-geist-sans);
  --font-mono: var(--font-geist-mono);
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}

body {
  background: var(--background);
  color: var(--foreground);
  font-family: Arial, Helvetica, sans-serif;
}
```

(Note: `body` is also given `bg-white text-neutral-900 dark:bg-neutral-950 dark:text-neutral-100` in
the layout. Geist is loaded and its variables are on `<html>`; keep both so the font vars exist
for `font-mono`.)

## Layout shell

```tsx
<html lang="en" className={`${geistSans.variable} ${geistMono.variable} h-full antialiased`}>
  <body className="min-h-full flex flex-col bg-white text-neutral-900 dark:bg-neutral-950 dark:text-neutral-100">
    <Nav />
    <main className="flex-1">{children}</main>
  </body>
</html>
```

Page containers: `mx-auto max-w-3xl px-4 py-12 sm:px-6` (home, history), `max-w-2xl px-4 py-10/12/16 sm:px-6`
(quiz, results). Nav inner container `max-w-4xl`. Chat is full-height: `h-[calc(100dvh-4rem)]`
(nav is `h-16`).

## Component recipes (copy these class strings)

**Nav** - `header.h-16.shrink-0.border-b.border-black/10.dark:border-white/10`; brand = violet dot
`inline-block h-2.5 w-2.5 rounded-full bg-violet-500` + app title `font-semibold tracking-tight`;
links `text-sm text-black/60 dark:text-white/60 transition hover:text-black dark:hover:text-white`.
Links: Practice, History, Ask AI.

**Page title** - `text-3xl font-bold tracking-tight`; subtitle `mt-2 text-neutral-600 dark:text-neutral-400`.

**Card** - `rounded-2xl border border-black/10 bg-neutral-50 p-6 dark:border-white/10 dark:bg-neutral-900`.

**Warning/in-progress card** - `rounded-2xl border border-amber-500/30 bg-amber-50 p-6 dark:border-amber-500/20 dark:bg-amber-950/20`.

**Primary button** - `rounded-lg bg-violet-600 px-4 py-3 text-sm font-semibold text-white transition hover:bg-violet-500`
(smaller: `px-5 py-2.5` or `px-3 py-2 text-xs`). Disabled: `disabled:cursor-not-allowed disabled:opacity-40`.

**Secondary button** - `rounded-lg border border-black/15 px-5 py-2.5 text-sm font-semibold transition hover:bg-black/5 dark:border-white/15 dark:hover:bg-white/5`.

**Destructive-on-hover button** - `rounded-lg border border-black/10 px-4 py-3 text-sm font-medium text-neutral-600 transition hover:border-red-400 hover:text-red-600 dark:border-white/10 dark:text-neutral-400 dark:hover:text-red-400`.

**Number input** - `w-32 rounded-lg border border-black/15 bg-white px-3 py-2 text-sm dark:border-white/15 dark:bg-neutral-950`.

**Domain checkbox row** - `label.flex.cursor-pointer.items-center.justify-between.gap-2.rounded-lg.border.border-black/10.bg-white.px-3.py-2.text-sm.transition.hover:border-violet-400.dark:border-white/10.dark:bg-neutral-950`; checkbox `accent-violet-600`; count `text-xs text-neutral-500`. Grid `grid-cols-1 gap-2 sm:grid-cols-2`.

**Progress bar** - track `h-2 w-full overflow-hidden rounded-full bg-neutral-200 dark:bg-neutral-800`, fill `h-full rounded-full bg-violet-600 transition-all` with inline `width: N%`.

**Pills** - domain: `rounded-full bg-violet-100 px-3 py-1 text-xs font-medium text-violet-700 dark:bg-violet-900/40 dark:text-violet-300`; type: `rounded-full bg-neutral-200 px-3 py-1 text-xs font-medium text-neutral-600 dark:bg-neutral-800 dark:text-neutral-300`.

**Answer option button** - base `flex items-center gap-3 rounded-xl border px-4 py-3 text-left text-sm transition`, plus state:
- idle: `border-black/10 dark:border-white/10 hover:border-violet-400`
- selected (pre-submit): `border-violet-500 bg-violet-50 dark:bg-violet-950/30`
- correct (post-submit): `border-emerald-500 bg-emerald-50 dark:bg-emerald-950/40`
- wrong selected: `border-red-500 bg-red-50 dark:bg-red-950/40`
- number badge: `flex h-6 w-6 flex-shrink-0 items-center justify-center rounded-full border border-current text-xs opacity-60`

**Feedback banner** - `mt-5 rounded-xl p-4 text-sm` +
correct `bg-emerald-100 text-emerald-800 dark:bg-emerald-950/50 dark:text-emerald-200`,
wrong `bg-red-100 text-red-800 dark:bg-red-950/50 dark:text-red-200`; heading `font-semibold` ("Correct!" / "Not quite."); explanation `text-neutral-700 dark:text-neutral-300`.

**Tables** - wrapper `overflow-hidden rounded-xl border border-black/10 dark:border-white/10`; `thead.bg-neutral-100.text-left.text-neutral-500.dark:bg-neutral-900`; `th px-4 py-3 font-medium`; `tr.border-t.border-black/10.dark:border-white/10`; `td px-4 py-3`.

**Score colour helper** (used for text and bars)
- text: `>=80 text-emerald-600 dark:text-emerald-400`, `>=60 text-amber-500`, else `text-red-500`
- bar fill: `bg-emerald-500 / bg-amber-500 / bg-red-500`

**Big score card** - card with `p-8 text-center`; label `text-sm font-medium uppercase tracking-wide text-neutral-500`; number `mt-2 text-5xl font-bold` + score colour.

**History row link** - `flex items-center justify-between rounded-xl border border-black/10 px-4 py-3 text-sm transition hover:border-violet-400 dark:border-white/10`.

**Chat bubbles** - user: `ml-auto max-w-[85%] rounded-2xl rounded-br-sm bg-violet-600 px-4 py-2.5 text-sm text-white`; assistant: `mr-auto max-w-[85%] rounded-2xl rounded-bl-sm bg-neutral-100 px-4 py-2.5 text-sm dark:bg-neutral-900`; error: `rounded-2xl rounded-bl-sm bg-red-100 px-4 py-2.5 text-xs text-red-700 dark:bg-red-950/50 dark:text-red-300`; thinking: assistant bubble + spinner + `text-xs text-neutral-500`.

**Chat sidebar** - `hidden w-64 shrink-0 flex-col border-r border-black/10 md:flex dark:border-white/10`; "+ New chat" = primary button full-width `block px-3 py-2 text-center text-sm`; conversation row `group flex items-center justify-between gap-2 rounded-lg px-3 py-2 text-sm transition`, active `bg-violet-100 text-violet-900 dark:bg-violet-900/40 dark:text-violet-100`, inactive `text-neutral-600 hover:bg-black/5 dark:text-neutral-300 dark:hover:bg-white/5`; delete "✕" `opacity-0 group-hover:opacity-100 hover:text-red-500`.

**Example prompt card (empty chat)** - `flex h-full flex-col gap-1.5 rounded-xl border border-black/10 bg-white p-3.5 text-left text-sm transition hover:border-violet-400 hover:shadow-sm disabled:cursor-not-allowed disabled:opacity-40 dark:border-white/10 dark:bg-neutral-950` with a tiny domain pill `text-[10px] px-2 py-0.5`.

**Chat input** - `flex-1 rounded-lg border border-black/10 bg-white px-3 py-2.5 text-sm outline-none focus:border-violet-400 dark:border-white/10 dark:bg-neutral-950`; textarea is `resize-none`, auto-grows to `maxRows`.

**Continue pill** - `rounded-full border border-violet-400 px-3.5 py-1.5 text-xs font-medium text-violet-700 transition hover:bg-violet-50 dark:text-violet-300 dark:hover:bg-violet-950/40`.

**"Explain further" link-button** - `text-xs font-medium text-violet-700 underline decoration-dotted hover:text-violet-500 dark:text-violet-300`.

**AI setup hint** - `rounded-xl border border-amber-500/30 bg-amber-50 p-4 text-sm text-amber-800 dark:bg-amber-950/30 dark:text-amber-200`.

**Keyboard hint footer** - `mt-4 text-center text-xs text-neutral-400`.

**Spinner** - svg 24x24, `className="h-4 w-4 animate-spin"`, circle stroke-opacity .25 + arc path, `strokeLinecap="round"`.

## Markdown rendering

Tailwind preflight strips heading sizes and list styles, so `Markdown.tsx` maps every element to
explicit classes (full file in `code-templates.md`). Code fences use `react-syntax-highlighter`
Prism with `oneDark`, `borderRadius: 0.75rem`, `padding: 1rem`, `fontSize: 0.85em`. Inline code:
`rounded bg-black/10 px-1 py-0.5 font-mono text-[0.85em] dark:bg-white/10`. A `tone="inverted"`
prop adapts links/quotes/inline-code for the solid violet user bubble.

## Changing the accent (optional)

The accent is only ever `violet-*`. If the user asks for another brand colour, search/replace the
`violet-` utility prefix app-wide (e.g. `indigo-`, `teal-`). Keep the same shade numbers. Default is
violet - do not change it unprompted; sameness across generated apps is intentional.

## Responsive & a11y

- Mobile first; sidebar is `hidden md:flex` (chat list hidden on small screens).
- Inputs have `<label htmlFor>`; delete button has `aria-label`.
- Don't rely on colour alone for answer state: correct/incorrect also change the feedback banner text ("Correct!"/"Not quite.").
- Keyboard: number keys select options, Enter submits/continues (ignored while typing in INPUT/TEXTAREA).
