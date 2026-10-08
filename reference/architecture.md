# Architecture

## Stack

| Concern | Choice |
|---|---|
| Framework | Next.js (App Router, TypeScript, `src/` dir, alias `@/*`) - React 19 |
| Styling | Tailwind CSS v4 (`@import "tailwindcss"`, `@tailwindcss/postcss`), no component library |
| Fonts | `next/font/google` Geist + Geist Mono, exposed as CSS vars |
| Persistence | Prisma 6 + SQLite (`file:./dev.db`), single user, no auth |
| Mutations | Server Actions (`"use server"`) - no REST API except `/api/ai` |
| AI | OpenAI Chat Completions via `fetch` in one route handler (`/api/ai`) - no SDK |
| Markdown | `react-markdown` + `remark-gfm` + `react-syntax-highlighter` (Prism, `oneDark`) |
| Questions | Static typed TypeScript data files (NOT in the DB) |

Why questions live in code, not the DB: they're version-controlled, type-checked, trivially
editable, and attempts only store *ids* + answers. A quiz "freezes" its question ids + option
order at start (`Attempt.questionIds` JSON), so editing the bank never corrupts history.

## Folder layout

```
<slug>/
  .env                      DATABASE_URL="file:./dev.db"
  .env.local                OPENAI_API_KEY=, OPENAI_MODEL=
  prisma/schema.prisma
  src/
    app/
      layout.tsx            fonts, <Nav/>, body theme classes
      globals.css           Tailwind import + light/dark tokens
      page.tsx              Home: start form / resume banner
      quiz/[attemptId]/{page.tsx,QuizRunner.tsx}
      results/[attemptId]/page.tsx
      history/page.tsx
      chat/{layout.tsx,page.tsx,ChatPanel.tsx,ConversationList.tsx}
      chat/[id]/page.tsx
      api/ai/route.ts
    components/{Nav,Markdown,MessageInput,DiscardAttemptButton}.tsx
    data/
      types.ts              DOMAINS, DOMAIN_LABELS, Question types
      examplePrompts.ts
      questions/index.ts + one file per domain
    lib/{prisma,quiz,actions,chatActions,stats}.ts
```

## Data model (Prisma)

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "sqlite"  url = env("DATABASE_URL") }

model Attempt {
  id             String    @id @default(cuid())
  startedAt      DateTime  @default(now())
  finishedAt     DateTime?
  totalQuestions Int
  correctCount   Int?
  scorePct       Float?
  domainFilter   String?   // JSON array of domain keys; null = all
  questionIds    String    // JSON PlannedQuestion[] frozen at start: [{questionId, optionOrder[]}]
  answers        AttemptAnswer[]
}

model AttemptAnswer {
  id                String   @id @default(cuid())
  attemptId         String
  attempt           Attempt  @relation(fields: [attemptId], references: [id], onDelete: Cascade)
  questionId        String
  domain            String
  questionType      String
  selectedOptionIds String   // JSON string[]
  isCorrect         Boolean
  answeredAt        DateTime @default(now())
  @@index([attemptId])
  @@index([domain])
}

model Conversation {
  id        String   @id @default(cuid())
  title     String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  messages  ChatMessage[]
}

model ChatMessage {
  id             String       @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  role           String       // "user" | "assistant"
  content        String
  truncated      Boolean      @default(false) // assistant reply hit the token cap
  createdAt      DateTime     @default(now())
  @@index([conversationId])
}
```

SQLite has no JSON/enum types - store arrays as JSON strings and `JSON.parse` on read.

## Behaviour spec (what "correct" means)

### Starting a quiz (`startAttempt`)
- Count defaults to 50 (min 5, max = pool size), parsed defensively (`Number.isFinite`, floor).
- Domains: zero checked = all domains.
- **Single active attempt:** if an unfinished `Attempt` exists, redirect to it instead of creating another (server-side guard so back-button/stale forms can't bypass the UI).
- `buildPlan`: filter pool by domain -> Fisher-Yates shuffle -> slice(count) -> for each question also shuffle option order. Persist plan JSON in `Attempt.questionIds`.
- `redirect()` to `/quiz/<id>`.

### Answering (`submitAnswer`)
- Looks up question in `ALL_QUESTIONS` server-side (client never receives correctness before submit... note: the full question incl. `correct` flags *is* serialised to the client component today; that's accepted for a single-user study tool. If you want to harden it, strip `correct` from the props and rely on the action's return value, which already carries `correctOptionIds`).
- Upsert one `AttemptAnswer` per (attempt, question) so re-submits don't duplicate.
- Grading is **all-or-nothing**: set of selected ids must equal set of correct ids.
- Returns `{ isCorrect, correctOptionIds, explanation }`.

### Resuming
- Quiz page loads answers already saved, computes `startIndex` = first question with no answer, and seeds the score from existing answers.

### Finishing (`finishAttempt`)
- Count correct answers, `scorePct = correct / totalQuestions * 100` (unanswered counts wrong), set `finishedAt`, redirect to `/results/<id>`.
- When the runner's index reaches `total`, auto-call `finishAttempt` from a `useEffect` inside `startTransition`.

### Discarding
- `discardAttempt` deletes **only unfinished** attempts (`deleteMany({id, finishedAt: null})`), then redirects `/`. UI confirms with `confirm()`.

### Weak areas (`lib/stats.ts`)
- `groupBy domain` totals and corrects across **all** `AttemptAnswer` rows; pct = round(correct/total*100); sorted ascending.
- `meaningfulWeakDomains(min=3)` = domains with >= 3 answers, weakest first.
- `weakAreaSummaryText` = up to 3 weakest meaningful domains rendered as `"<Label> (<pct>% accuracy over <n> questions)"` joined by `; ` - injected into the chat system prompt so the AI tutor knows the learner's gaps.

### Colour thresholds (everywhere a score/pct is shown)
- `>= 80` emerald, `>= 60` amber, `< 60` red.

### Routes

| Route | Type | Notes |
|---|---|---|
| `/` | server, `force-dynamic` | counts per domain from the bank; active-attempt banner |
| `/quiz/[attemptId]` | server page + client `QuizRunner` | redirects to results if finished; 404 if missing |
| `/results/[attemptId]` | server | redirects to quiz if unfinished |
| `/history` | server | weak areas + last 50 finished attempts |
| `/chat` | server layout (sidebar of conversations) + client `ChatPanel` | shows setup hint if no API key |
| `/chat/[id]` | server | loads saved messages |
| `/api/ai` | POST route handler | 501 if no key; 400 invalid; 502 upstream |

Everything that reads the DB uses `export const dynamic = "force-dynamic"`.

## Next.js specifics to respect (verify against installed docs)

- `params` in pages/layouts is a **Promise** - `const { attemptId } = await params;`
- Root layout typed with the global `LayoutProps<"/">` helper (generated by Next's typegen). If it isn't available in the installed version, fall back to `{ children: React.ReactNode }`.
- Server Actions live in files starting with `"use server"`; client components import and call them directly (wrap in `useTransition`).
- `revalidatePath("/chat")` after chat mutations so the sidebar refreshes.
- `redirect()` throws - never wrap it in try/catch.
- Client components that call server actions must not import `prisma` anywhere.
