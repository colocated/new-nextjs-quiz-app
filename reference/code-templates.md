# Code templates

Complete source for every non-content file. Substitute the `APP_CONFIG` values (`src/data/app.ts`)
and the domain list; everything else is subject-agnostic. If the installed Next.js/Tailwind/Prisma
differs from these templates, keep behaviour + classes and adjust syntax to the installed docs.

Order to write: config -> prisma -> lib -> data -> components -> app routes -> API -> README.

---

## `src/data/app.ts` (the only per-app config besides domains)

```ts
export const APP_CONFIG = {
  title: "{{APP_TITLE}}",                       // e.g. "Java Engineering Course"
  navTitle: "{{APP_TITLE}}",                    // short form for the nav bar
  tagline: "{{ONE_LINE_TAGLINE}}",              // under the home H1
  subject: "{{SUBJECT}}",                       // e.g. "Java and the JVM ecosystem"
  tutorRole: "{{TUTOR_ROLE}}",                  // e.g. "a senior Java engineer and interview coach"
  learnerContext: "{{LEARNER_CONTEXT}}",        // 1-2 sentences, NO personal data. "" if none
  codeFence: "{{CODE_LANG}}",                   // "java", "hcl", "python", "sql" ... or "text"
  askAiNoun: "{{SUBJECT_SHORT}}",               // "Java" -> "Ask about Java"
};
```

## `src/data/types.ts`

```ts
// Generated from the plan: 6-12 domains. Keys are kebab-case ids used in question ids.
export const DOMAINS = [
  // "core-language",
  // "concurrency",
] as const;

export type Domain = (typeof DOMAINS)[number];

export const DOMAIN_LABELS: Record<Domain, string> = {
  // "core-language": "Core Language",
  // "concurrency": "Concurrency",
};

export type QuestionType = "single" | "multi" | "true-false";

export interface QuestionOption {
  id: string;
  text: string;
  correct: boolean;
}

export interface Question {
  id: string;
  domain: Domain;
  type: QuestionType;
  prompt: string;
  options: QuestionOption[];
  explanation: string;
}
```

## `src/data/questions/index.ts`

```ts
import { Question } from "../types";
import { coreLanguageQuestions } from "./core-language";
// ...one import per domain file

export const ALL_QUESTIONS: Question[] = [
  ...coreLanguageQuestions,
  // ...
];

// Guard against accidental duplicate IDs across domain files.
const seen = new Set<string>();
for (const q of ALL_QUESTIONS) {
  if (seen.has(q.id)) {
    throw new Error(`Duplicate question id detected: ${q.id}`);
  }
  seen.add(q.id);
}
```

## `src/data/examplePrompts.ts`

```ts
import { DOMAINS, Domain } from "./types";

// Hand-written - one honest question per domain a learner would actually want to ask.
export const EXAMPLE_PROMPTS: Record<Domain, string[]> = {
  // "core-language": [
  //   "What's the practical difference between X and Y?",
  //   "Walk me through what happens when Z.",
  // ],
};

export interface ExamplePrompt {
  domain: Domain;
  prompt: string;
}

// Weakest domains first (up to 2 prompts from the very weakest), then pads
// with untouched domains in canonical order so it's never a short list.
export function pickExamplePrompts(weakDomainsWeakestFirst: Domain[], max = 4): ExamplePrompt[] {
  const picked: ExamplePrompt[] = [];
  const usedDomains = new Set<Domain>();

  function addFrom(domain: Domain, count: number) {
    if (usedDomains.has(domain)) return;
    usedDomains.add(domain);
    for (const prompt of (EXAMPLE_PROMPTS[domain] ?? []).slice(0, count)) {
      if (picked.length >= max) return;
      picked.push({ domain, prompt });
    }
  }

  weakDomainsWeakestFirst.forEach((domain, i) => {
    if (picked.length < max) addFrom(domain, i === 0 ? 2 : 1);
  });

  for (const domain of DOMAINS) {
    if (picked.length >= max) break;
    addFrom(domain, 1);
  }

  return picked.slice(0, max);
}
```

## `prisma/schema.prisma`

See `architecture.md` (copy the 4 models verbatim). Then:

```bash
echo 'DATABASE_URL="file:./dev.db"' > .env
npx prisma migrate dev --name init
```

## `src/lib/prisma.ts`

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["warn", "error"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

## `src/lib/quiz.ts`

```ts
import { ALL_QUESTIONS } from "@/data/questions";
import { Domain, Question } from "@/data/types";

export interface PlannedQuestion {
  questionId: string;
  optionOrder: string[];
}

function shuffle<T>(items: T[]): T[] {
  const arr = [...items];
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

export function buildPlan(count: number, domains: Domain[] | null): PlannedQuestion[] {
  const pool = domains && domains.length > 0
    ? ALL_QUESTIONS.filter((q) => domains.includes(q.domain))
    : ALL_QUESTIONS;

  const size = Math.min(count, pool.length);
  const selected = shuffle(pool).slice(0, size);

  return selected.map((q) => ({
    questionId: q.id,
    optionOrder: shuffle(q.options.map((o) => o.id)),
  }));
}

export interface OrderedQuestion extends Question {
  orderedOptionIds: string[];
}

export function hydratePlan(plan: PlannedQuestion[]): OrderedQuestion[] {
  const byId = new Map(ALL_QUESTIONS.map((q) => [q.id, q]));
  return plan
    .map((p) => {
      const q = byId.get(p.questionId);
      if (!q) return null;
      return { ...q, orderedOptionIds: p.optionOrder };
    })
    .filter((q): q is OrderedQuestion => q !== null);
}

export function isAnswerCorrect(question: Question, selectedOptionIds: string[]): boolean {
  const correctIds = new Set(question.options.filter((o) => o.correct).map((o) => o.id));
  const selectedSet = new Set(selectedOptionIds);
  if (correctIds.size !== selectedSet.size) return false;
  for (const id of correctIds) {
    if (!selectedSet.has(id)) return false;
  }
  return true;
}
```

> Enhancement allowed (not in the original): true-false options keep a fixed order ("True" first) - if
> you want that, skip shuffling `optionOrder` when `q.type === "true-false"`. Default: shuffle all.

## `src/lib/actions.ts`

```ts
"use server";

import { redirect } from "next/navigation";
import { prisma } from "@/lib/prisma";
import { buildPlan, hydratePlan, isAnswerCorrect, PlannedQuestion } from "@/lib/quiz";
import { ALL_QUESTIONS } from "@/data/questions";
import { Domain } from "@/data/types";

export async function startAttemptFromForm(formData: FormData) {
  const rawCount = Number(formData.get("count"));
  const count = Number.isFinite(rawCount) && rawCount > 0 ? Math.floor(rawCount) : 50;
  const domains = formData.getAll("domains").map((d) => d.toString()) as Domain[];
  await startAttempt(count, domains.length > 0 ? domains : null);
}

export async function discardAttempt(formData: FormData) {
  const attemptId = formData.get("attemptId")?.toString();
  if (!attemptId) return;
  // Unfinished attempts only - never lets someone delete a completed, scored run.
  await prisma.attempt.deleteMany({ where: { id: attemptId, finishedAt: null } });
  redirect("/");
}

export async function startAttempt(count: number, domains: Domain[] | null) {
  // Server-side guard against a stale/cached form (e.g. browser back button)
  // bypassing the "resume or discard" screen on /.
  const active = await prisma.attempt.findFirst({ where: { finishedAt: null } });
  if (active) {
    redirect(`/quiz/${active.id}`);
  }

  const plan = buildPlan(count, domains);
  if (plan.length === 0) {
    throw new Error("No questions match the selected filters.");
  }

  const attempt = await prisma.attempt.create({
    data: {
      totalQuestions: plan.length,
      domainFilter: domains && domains.length > 0 ? JSON.stringify(domains) : null,
      questionIds: JSON.stringify(plan),
    },
  });

  redirect(`/quiz/${attempt.id}`);
}

export interface SubmitAnswerResult {
  isCorrect: boolean;
  correctOptionIds: string[];
  explanation: string;
}

export async function submitAnswer(
  attemptId: string,
  questionId: string,
  selectedOptionIds: string[]
): Promise<SubmitAnswerResult> {
  const question = ALL_QUESTIONS.find((q) => q.id === questionId);
  if (!question) throw new Error("Unknown question");

  const existing = await prisma.attemptAnswer.findFirst({
    where: { attemptId, questionId },
  });

  const correct = isAnswerCorrect(question, selectedOptionIds);

  if (existing) {
    await prisma.attemptAnswer.update({
      where: { id: existing.id },
      data: {
        selectedOptionIds: JSON.stringify(selectedOptionIds),
        isCorrect: correct,
      },
    });
  } else {
    await prisma.attemptAnswer.create({
      data: {
        attemptId,
        questionId,
        domain: question.domain,
        questionType: question.type,
        selectedOptionIds: JSON.stringify(selectedOptionIds),
        isCorrect: correct,
      },
    });
  }

  return {
    isCorrect: correct,
    correctOptionIds: question.options.filter((o) => o.correct).map((o) => o.id),
    explanation: question.explanation,
  };
}

export async function finishAttempt(attemptId: string) {
  const answers = await prisma.attemptAnswer.findMany({ where: { attemptId } });
  const correctCount = answers.filter((a) => a.isCorrect).length;
  const attempt = await prisma.attempt.findUniqueOrThrow({ where: { id: attemptId } });
  const scorePct = attempt.totalQuestions > 0 ? (correctCount / attempt.totalQuestions) * 100 : 0;

  await prisma.attempt.update({
    where: { id: attemptId },
    data: {
      finishedAt: new Date(),
      correctCount,
      scorePct,
    },
  });

  redirect(`/results/${attemptId}`);
}

export async function getAttemptWithQuestions(attemptId: string) {
  const attempt = await prisma.attempt.findUnique({ where: { id: attemptId } });
  if (!attempt) return null;
  const plan: PlannedQuestion[] = JSON.parse(attempt.questionIds);
  const questions = hydratePlan(plan);
  return { attempt, questions };
}
```

> Note: `getAttemptWithQuestions` is exported from a `"use server"` file, which makes it callable as
> a server action. It is only used from server components here. If you prefer, move it to
> `lib/quiz.ts`-adjacent server-only module (no `"use server"`).

## `src/lib/chatActions.ts`

```ts
"use server";

import { revalidatePath } from "next/cache";
import { prisma } from "@/lib/prisma";

export type ChatRole = "user" | "assistant";

function titleFrom(content: string): string {
  const oneLine = content.replace(/\s+/g, " ").trim();
  return oneLine.length > 60 ? `${oneLine.slice(0, 57)}...` : oneLine;
}

export async function createConversationWithMessage(content: string): Promise<string> {
  const conversation = await prisma.conversation.create({
    data: {
      title: titleFrom(content),
      messages: { create: { role: "user", content } },
    },
  });
  revalidatePath("/chat");
  return conversation.id;
}

export async function appendMessage(
  conversationId: string,
  role: ChatRole,
  content: string,
  truncated = false
) {
  await prisma.conversation.update({
    where: { id: conversationId },
    data: {
      updatedAt: new Date(),
      messages: { create: { role, content, truncated } },
    },
  });
  revalidatePath("/chat");
}

// Used by the Continue button: appends the continuation to the most recent
// assistant message rather than creating a new one.
export async function extendLastAssistantMessage(
  conversationId: string,
  extra: string,
  truncated: boolean
) {
  const last = await prisma.chatMessage.findFirst({
    where: { conversationId, role: "assistant" },
    orderBy: { createdAt: "desc" },
  });
  if (!last) return;
  await prisma.chatMessage.update({
    where: { id: last.id },
    data: { content: last.content + extra, truncated },
  });
  await prisma.conversation.update({
    where: { id: conversationId },
    data: { updatedAt: new Date() },
  });
  revalidatePath("/chat");
}

export async function listConversations() {
  return prisma.conversation.findMany({
    orderBy: { updatedAt: "desc" },
    select: { id: true, title: true, updatedAt: true },
  });
}

export async function getConversation(id: string) {
  return prisma.conversation.findUnique({
    where: { id },
    include: { messages: { orderBy: { createdAt: "asc" } } },
  });
}

export async function deleteConversation(id: string) {
  await prisma.conversation.delete({ where: { id } });
  revalidatePath("/chat");
}
```

## `src/lib/stats.ts`

```ts
import { prisma } from "@/lib/prisma";
import { Domain, DOMAIN_LABELS } from "@/data/types";

export interface WeakArea {
  domain: Domain;
  total: number;
  correct: number;
  pct: number;
}

export async function getWeakAreas(): Promise<WeakArea[]> {
  const totals = await prisma.attemptAnswer.groupBy({
    by: ["domain"],
    _count: { _all: true },
  });
  const corrects = await prisma.attemptAnswer.groupBy({
    by: ["domain"],
    where: { isCorrect: true },
    _count: { _all: true },
  });
  const correctByDomain = new Map(corrects.map((c) => [c.domain, c._count._all]));

  return totals
    .map((t) => {
      const total = t._count._all;
      const correct = correctByDomain.get(t.domain) ?? 0;
      return {
        domain: t.domain as Domain,
        total,
        correct,
        pct: total > 0 ? Math.round((correct / total) * 100) : 0,
      };
    })
    .sort((a, b) => a.pct - b.pct);
}

// Domains with enough answered questions to be a meaningful signal, weakest first.
export function meaningfulWeakDomains(weakAreas: WeakArea[], minAnswered = 3): Domain[] {
  return weakAreas.filter((w) => w.total >= minAnswered).map((w) => w.domain);
}

export function weakAreaSummaryText(weakAreas: WeakArea[], minAnswered = 3): string {
  return weakAreas
    .filter((w) => w.total >= minAnswered)
    .slice(0, 3)
    .map((w) => `${DOMAIN_LABELS[w.domain]} (${w.pct}% accuracy over ${w.total} questions)`)
    .join("; ");
}
```

> Guard: a stored `domain` that no longer exists in `DOMAIN_LABELS` (after you rename a domain) would
> render `undefined`. Render `DOMAIN_LABELS[d] ?? d` in tables if you expect domain churn.

---

## `src/components/Nav.tsx`

```tsx
import Link from "next/link";
import { APP_CONFIG } from "@/data/app";

export function Nav() {
  return (
    <header className="h-16 shrink-0 border-b border-black/10 dark:border-white/10">
      <div className="mx-auto flex h-full max-w-4xl items-center justify-between px-4 sm:px-6">
        <Link href="/" className="flex items-center gap-2 font-semibold tracking-tight">
          <span className="inline-block h-2.5 w-2.5 rounded-full bg-violet-500" />
          {APP_CONFIG.navTitle}
        </Link>
        <nav className="flex items-center gap-5 text-sm text-black/60 dark:text-white/60">
          <Link href="/" className="transition hover:text-black dark:hover:text-white">
            Practice
          </Link>
          <Link href="/history" className="transition hover:text-black dark:hover:text-white">
            History
          </Link>
          <Link href="/chat" className="transition hover:text-black dark:hover:text-white">
            Ask AI
          </Link>
        </nav>
      </div>
    </header>
  );
}
```

## `src/components/DiscardAttemptButton.tsx`

```tsx
"use client";

import { FormEvent } from "react";
import { discardAttempt } from "@/lib/actions";

export function DiscardAttemptButton({ attemptId }: { attemptId: string }) {
  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    if (!confirm("Discard this in-progress attempt? Its answers so far will be deleted.")) {
      e.preventDefault();
    }
  }

  return (
    <form action={discardAttempt} onSubmit={handleSubmit}>
      <input type="hidden" name="attemptId" value={attemptId} />
      <button
        type="submit"
        className="rounded-lg border border-black/10 px-4 py-3 text-sm font-medium text-neutral-600 transition hover:border-red-400 hover:text-red-600 dark:border-white/10 dark:text-neutral-400 dark:hover:text-red-400"
      >
        Discard &amp; start new
      </button>
    </form>
  );
}
```

## `src/components/MessageInput.tsx`

```tsx
"use client";

import {
  KeyboardEvent,
  TextareaHTMLAttributes,
  forwardRef,
  useEffect,
  useImperativeHandle,
  useRef,
} from "react";

interface MessageInputProps
  extends Omit<TextareaHTMLAttributes<HTMLTextAreaElement>, "value" | "onChange" | "onKeyDown"> {
  value: string;
  onChange: (value: string) => void;
  onSubmit: () => void;
  maxRows?: number;
}

// Plain multiline textarea, not a live-rendering rich-text editor: Enter
// sends, Shift+Enter inserts a newline, and it grows with content (a pasted
// code block included) up to maxRows before it starts scrolling internally.
export const MessageInput = forwardRef<HTMLTextAreaElement, MessageInputProps>(function MessageInput(
  { value, onChange, onSubmit, maxRows = 8, className, ...rest },
  forwardedRef
) {
  const ref = useRef<HTMLTextAreaElement>(null);
  useImperativeHandle(forwardedRef, () => ref.current as HTMLTextAreaElement);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    el.style.height = "auto";
    const lineHeight = parseFloat(getComputedStyle(el).lineHeight || "20") || 20;
    const maxHeight = lineHeight * maxRows;
    el.style.height = `${Math.min(el.scrollHeight, maxHeight)}px`;
    el.style.overflowY = el.scrollHeight > maxHeight ? "auto" : "hidden";
  }, [value, maxRows]);

  function handleKeyDown(e: KeyboardEvent<HTMLTextAreaElement>) {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      onSubmit();
    }
  }

  return (
    <textarea
      ref={ref}
      value={value}
      onChange={(e) => onChange(e.target.value)}
      onKeyDown={handleKeyDown}
      rows={1}
      className={`resize-none ${className ?? ""}`}
      {...rest}
    />
  );
});
```

## `src/components/Markdown.tsx`

```tsx
import ReactMarkdown from "react-markdown";
import remarkGfm from "remark-gfm";
import { Prism as SyntaxHighlighter } from "react-syntax-highlighter";
import { oneDark } from "react-syntax-highlighter/dist/esm/styles/prism";

// Tailwind's preflight reset strips default font-size/weight from headings and
// list-style from ul/ol, so every element markdown can produce needs an
// explicit class here - there's no typography plugin doing it for us.
//
// `tone="inverted"` is for rendering on a solid color-600 bubble (e.g. a
// user's own message) where the default violet link/blockquote colors would
// have poor contrast against the background.
export function Markdown({ content, tone = "default" }: { content: string; tone?: "default" | "inverted" }) {
  return (
    <div className="max-w-none">
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        components={{
          h1: (props) => <h1 className="mb-2 mt-4 text-lg font-bold first:mt-0" {...props} />,
          h2: (props) => <h2 className="mb-2 mt-3 text-base font-bold first:mt-0" {...props} />,
          h3: (props) => <h3 className="mb-1 mt-3 text-sm font-bold first:mt-0" {...props} />,
          h4: (props) => <h4 className="mb-1 mt-2 text-sm font-semibold first:mt-0" {...props} />,
          h5: (props) => <h5 className="mb-1 mt-2 text-sm font-semibold first:mt-0" {...props} />,
          h6: (props) => <h6 className="mb-1 mt-2 text-sm font-semibold first:mt-0" {...props} />,
          p: (props) => <p className="my-2 first:mt-0 last:mb-0" {...props} />,
          ul: (props) => <ul className="my-2 list-disc space-y-0.5 pl-5" {...props} />,
          ol: (props) => <ol className="my-2 list-decimal space-y-0.5 pl-5" {...props} />,
          li: (props) => <li className="marker:text-neutral-400" {...props} />,
          strong: (props) => <strong className="font-semibold" {...props} />,
          blockquote: (props) => (
            <blockquote
              className={
                tone === "inverted"
                  ? "my-2 border-l-2 border-white/40 pl-3 italic text-white/85"
                  : "my-2 border-l-2 border-violet-400 pl-3 italic text-neutral-600 dark:border-violet-500 dark:text-neutral-400"
              }
              {...props}
            />
          ),
          hr: (props) => <hr className="my-4 border-black/10 dark:border-white/10" {...props} />,
          table: (props) => (
            <div className="my-3 overflow-x-auto">
              <table className="w-full border-collapse text-xs" {...props} />
            </div>
          ),
          thead: (props) => <thead className="bg-black/5 dark:bg-white/10" {...props} />,
          th: (props) => (
            <th className="border border-black/10 px-2 py-1 text-left font-semibold dark:border-white/10" {...props} />
          ),
          td: (props) => <td className="border border-black/10 px-2 py-1 dark:border-white/10" {...props} />,
          // react-syntax-highlighter's Prism renders its own <pre>, so `pre`
          // below is just a pass-through - wrapping it in another <pre> here
          // would nest them.
          code(props) {
            const { className, children } = props;
            const match = /language-(\w+)/.exec(className ?? "");
            if (!match) {
              return (
                <code
                  className={
                    tone === "inverted"
                      ? "rounded bg-black/15 px-1 py-0.5 font-mono text-[0.85em]"
                      : "rounded bg-black/10 px-1 py-0.5 font-mono text-[0.85em] dark:bg-white/10"
                  }
                >
                  {children}
                </code>
              );
            }
            const codeString = String(children).replace(/\n$/, "");
            return (
              <SyntaxHighlighter
                language={match[1]}
                style={oneDark}
                customStyle={{
                  margin: "0.75rem 0",
                  borderRadius: "0.75rem",
                  padding: "1rem",
                  fontSize: "0.85em",
                }}
              >
                {codeString}
              </SyntaxHighlighter>
            );
          },
          pre: (props) => <>{props.children}</>,
          a(props) {
            return (
              <a
                {...props}
                target="_blank"
                rel="noopener noreferrer"
                className={
                  tone === "inverted"
                    ? "text-white underline decoration-white/50 hover:decoration-white"
                    : "text-violet-600 underline dark:text-violet-400"
                }
              />
            );
          },
        }}
      >
        {content}
      </ReactMarkdown>
    </div>
  );
}
```

> Markdown from the model is untrusted: `react-markdown` does not render raw HTML by default - do NOT add
> `rehype-raw`. Question prompts/options are plain text; if you want code in a question prompt, render
> the prompt through `<Markdown>` too (see question-bank.md "Code in questions").

---

## `src/app/layout.tsx`

```tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import { Nav } from "@/components/Nav";
import { APP_CONFIG } from "@/data/app";
import "./globals.css";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: APP_CONFIG.title,
  description: APP_CONFIG.tagline,
};

export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html
      lang="en"
      className={`${geistSans.variable} ${geistMono.variable} h-full antialiased`}
    >
      <body className="min-h-full flex flex-col bg-white text-neutral-900 dark:bg-neutral-950 dark:text-neutral-100">
        <Nav />
        <main className="flex-1">{children}</main>
      </body>
    </html>
  );
}
```

`globals.css`: see `design-system.md`.

## `src/app/page.tsx`

```tsx
import Link from "next/link";
import { ALL_QUESTIONS } from "@/data/questions";
import { DOMAINS, DOMAIN_LABELS } from "@/data/types";
import { APP_CONFIG } from "@/data/app";
import { startAttemptFromForm } from "@/lib/actions";
import { prisma } from "@/lib/prisma";
import { DiscardAttemptButton } from "@/components/DiscardAttemptButton";

export const dynamic = "force-dynamic";

export default async function Home() {
  const domainCounts = DOMAINS.map((domain) => ({
    domain,
    label: DOMAIN_LABELS[domain],
    count: ALL_QUESTIONS.filter((q) => q.domain === domain).length,
  }));

  const total = ALL_QUESTIONS.length;

  const activeAttempt = await prisma.attempt.findFirst({
    where: { finishedAt: null },
    orderBy: { startedAt: "desc" },
    include: { _count: { select: { answers: true } } },
  });

  return (
    <div className="mx-auto max-w-3xl px-4 py-12 sm:px-6">
      <div className="mb-10">
        <h1 className="text-3xl font-bold tracking-tight">{APP_CONFIG.title}</h1>
        <p className="mt-2 text-neutral-600 dark:text-neutral-400">
          {APP_CONFIG.tagline} A fresh, randomized set of questions every time — {total} questions in
          the bank across {DOMAINS.length} domains. Scores and weak areas are tracked across every attempt.
        </p>
      </div>

      {activeAttempt ? (
        <div className="rounded-2xl border border-amber-500/30 bg-amber-50 p-6 dark:border-amber-500/20 dark:bg-amber-950/20">
          <h2 className="mb-1 text-lg font-semibold">You have a quiz in progress</h2>
          <p className="mb-5 text-sm text-neutral-600 dark:text-neutral-400">
            Started {activeAttempt.startedAt.toLocaleString()} &middot;{" "}
            {activeAttempt._count.answers}/{activeAttempt.totalQuestions} answered. Finish or
            discard it before starting a new one.
          </p>
          <div className="flex flex-wrap gap-3">
            <Link
              href={`/quiz/${activeAttempt.id}`}
              className="rounded-lg bg-violet-600 px-4 py-3 text-sm font-semibold text-white transition hover:bg-violet-500"
            >
              Resume quiz
            </Link>
            <DiscardAttemptButton attemptId={activeAttempt.id} />
          </div>
        </div>
      ) : (
        <form
          action={startAttemptFromForm}
          className="rounded-2xl border border-black/10 bg-neutral-50 p-6 dark:border-white/10 dark:bg-neutral-900"
        >
          <div className="mb-6">
            <label htmlFor="count" className="mb-2 block text-sm font-medium">
              Number of questions
            </label>
            <input
              id="count"
              name="count"
              type="number"
              min={5}
              max={total}
              defaultValue={Math.min(50, total)}
              className="w-32 rounded-lg border border-black/15 bg-white px-3 py-2 text-sm dark:border-white/15 dark:bg-neutral-950"
            />
            <span className="ml-3 text-sm text-neutral-500">of {total} available</span>
          </div>

          <div className="mb-6">
            <p className="mb-3 text-sm font-medium">
              Domains{" "}
              <span className="font-normal text-neutral-500">
                (leave all unchecked to include every domain)
              </span>
            </p>
            <div className="grid grid-cols-1 gap-2 sm:grid-cols-2">
              {domainCounts.map(({ domain, label, count }) => (
                <label
                  key={domain}
                  className="flex cursor-pointer items-center justify-between gap-2 rounded-lg border border-black/10 bg-white px-3 py-2 text-sm transition hover:border-violet-400 dark:border-white/10 dark:bg-neutral-950"
                >
                  <span className="flex items-center gap-2">
                    <input type="checkbox" name="domains" value={domain} className="accent-violet-600" />
                    {label}
                  </span>
                  <span className="text-xs text-neutral-500">{count}</span>
                </label>
              ))}
            </div>
          </div>

          <button
            type="submit"
            className="w-full rounded-lg bg-violet-600 px-4 py-3 text-sm font-semibold text-white transition hover:bg-violet-500"
          >
            Start Quiz
          </button>
        </form>
      )}
    </div>
  );
}
```

## `src/app/quiz/[attemptId]/page.tsx`

```tsx
import { notFound, redirect } from "next/navigation";
import { getAttemptWithQuestions } from "@/lib/actions";
import { prisma } from "@/lib/prisma";
import { QuizRunner } from "./QuizRunner";

export const dynamic = "force-dynamic";

export default async function QuizPage({
  params,
}: {
  params: Promise<{ attemptId: string }>;
}) {
  const { attemptId } = await params;
  const data = await getAttemptWithQuestions(attemptId);
  if (!data) notFound();

  const { attempt, questions } = data;

  if (attempt.finishedAt) {
    redirect(`/results/${attempt.id}`);
  }

  const existingAnswers = await prisma.attemptAnswer.findMany({
    where: { attemptId: attempt.id },
  });
  const answeredIds = new Set(existingAnswers.map((a) => a.questionId));
  const firstUnanswered = questions.findIndex((q) => !answeredIds.has(q.id));
  const startIndex = firstUnanswered === -1 ? questions.length : firstUnanswered;

  return (
    <QuizRunner
      attemptId={attempt.id}
      questions={questions}
      startIndex={startIndex}
      initialCorrectCount={existingAnswers.filter((a) => a.isCorrect).length}
      aiExplainEnabled={Boolean(process.env.OPENAI_API_KEY)}
    />
  );
}
```

## `src/app/quiz/[attemptId]/QuizRunner.tsx`

```tsx
"use client";

import { FormEvent, useEffect, useMemo, useState, useTransition } from "react";
import { APP_CONFIG } from "@/data/app";
import { DOMAIN_LABELS } from "@/data/types";
import { OrderedQuestion } from "@/lib/quiz";
import { finishAttempt, submitAnswer, SubmitAnswerResult } from "@/lib/actions";
import { Markdown } from "@/components/Markdown";
import { MessageInput } from "@/components/MessageInput";

interface AiMessage {
  role: "user" | "assistant";
  content: string;
}

interface QuizRunnerProps {
  attemptId: string;
  questions: OrderedQuestion[];
  startIndex: number;
  initialCorrectCount: number;
  aiExplainEnabled: boolean;
}

export function QuizRunner({
  attemptId,
  questions,
  startIndex,
  initialCorrectCount,
  aiExplainEnabled,
}: QuizRunnerProps) {
  const total = questions.length;
  const [index, setIndex] = useState(Math.min(startIndex, total));
  const [selected, setSelected] = useState<string[]>([]);
  const [submitted, setSubmitted] = useState(false);
  const [feedback, setFeedback] = useState<SubmitAnswerResult | null>(null);
  const [correctCount, setCorrectCount] = useState(initialCorrectCount);
  const [aiThread, setAiThread] = useState<AiMessage[]>([]);
  const [aiLoading, setAiLoading] = useState(false);
  const [aiFollowUp, setAiFollowUp] = useState("");
  const [pending, startTransition] = useTransition();

  const question = index < total ? questions[index] : null;

  const orderedOptions = useMemo(() => {
    if (!question) return [];
    return question.orderedOptionIds
      .map((id) => question.options.find((o) => o.id === id))
      .filter((o): o is NonNullable<typeof o> => Boolean(o));
  }, [question]);

  useEffect(() => {
    if (index >= total && total > 0) {
      startTransition(() => {
        finishAttempt(attemptId);
      });
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [index, total]);

  function toggleOption(optionId: string) {
    if (submitted || !question) return;
    if (question.type === "multi") {
      setSelected((prev) =>
        prev.includes(optionId) ? prev.filter((id) => id !== optionId) : [...prev, optionId]
      );
    } else {
      setSelected([optionId]);
    }
  }

  function handleSubmit() {
    if (!question || selected.length === 0 || submitted) return;
    startTransition(async () => {
      const result = await submitAnswer(attemptId, question.id, selected);
      setFeedback(result);
      setSubmitted(true);
      if (result.isCorrect) setCorrectCount((c) => c + 1);
    });
  }

  function handleNext() {
    setSelected([]);
    setSubmitted(false);
    setFeedback(null);
    setAiThread([]);
    setAiFollowUp("");
    setIndex((i) => i + 1);
  }

  function questionSystemPrompt() {
    if (!question) return "";
    return [
      `You are ${APP_CONFIG.tutorRole} helping a student who just answered a practice question about ${APP_CONFIG.subject}.`,
      APP_CONFIG.learnerContext ? `Learner context: ${APP_CONFIG.learnerContext}` : "",
      `Domain: ${DOMAIN_LABELS[question.domain]}`,
      `Question: ${question.prompt}`,
      `Reference explanation: ${question.explanation}`,
      `Give short, practical answers (a few sentences, or a brief code block in a \`\`\`${APP_CONFIG.codeFence} fence when a sample genuinely helps). Stay focused on this question and concepts related to it. If the student asks a follow-up, address it directly rather than repeating the whole explanation.`,
    ]
      .filter(Boolean)
      .join("\n");
  }

  async function askAi(userContent: string, threadSoFar: AiMessage[]) {
    setAiLoading(true);
    try {
      const res = await fetch("/api/ai", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          messages: [
            { role: "system", content: questionSystemPrompt() },
            ...threadSoFar,
            { role: "user", content: userContent },
          ],
        }),
      });
      const data = await res.json();
      const next: AiMessage[] = [
        ...threadSoFar,
        { role: "user", content: userContent },
        { role: "assistant", content: res.ok ? data.content : `(${data.error ?? "Something went wrong."})` },
      ];
      setAiThread(next);
    } catch {
      setAiThread([
        ...threadSoFar,
        { role: "user", content: userContent },
        { role: "assistant", content: "(Failed to reach the AI explainer.)" },
      ]);
    } finally {
      setAiLoading(false);
    }
  }

  function handleExplainFurther() {
    askAi("Give a deeper, more practical explanation of this question and its answer.", []);
  }

  function handleFollowUp(e: FormEvent) {
    e.preventDefault();
    submitFollowUp();
  }

  function submitFollowUp() {
    const content = aiFollowUp.trim();
    if (!content || aiLoading) return;
    setAiFollowUp("");
    askAi(content, aiThread);
  }

  useEffect(() => {
    function onKeyDown(e: KeyboardEvent) {
      if (!question) return;
      const target = e.target as HTMLElement | null;
      if (target && (target.tagName === "INPUT" || target.tagName === "TEXTAREA")) return;
      if (e.key === "Enter") {
        e.preventDefault();
        if (!submitted) handleSubmit();
        else handleNext();
        return;
      }
      if (!submitted) {
        const n = Number(e.key);
        if (n >= 1 && n <= orderedOptions.length) {
          toggleOption(orderedOptions[n - 1].id);
        }
      }
    }
    window.addEventListener("keydown", onKeyDown);
    return () => window.removeEventListener("keydown", onKeyDown);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [question, submitted, selected, orderedOptions]);

  if (!question) {
    return (
      <div className="mx-auto max-w-2xl px-4 py-16 text-center text-neutral-500">
        Finishing up...
      </div>
    );
  }

  const progressPct = Math.round((index / total) * 100);

  return (
    <div className="mx-auto max-w-2xl px-4 py-10 sm:px-6">
      <div className="mb-6">
        <div className="mb-2 flex items-center justify-between text-sm text-neutral-500">
          <span>
            Question {index + 1} of {total}
          </span>
          <span>
            Score: {correctCount}/{index + (submitted ? 1 : 0)}
          </span>
        </div>
        <div className="h-2 w-full overflow-hidden rounded-full bg-neutral-200 dark:bg-neutral-800">
          <div
            className="h-full rounded-full bg-violet-600 transition-all"
            style={{ width: `${progressPct}%` }}
          />
        </div>
      </div>

      <div className="rounded-2xl border border-black/10 bg-neutral-50 p-6 dark:border-white/10 dark:bg-neutral-900">
        <div className="mb-4 flex items-center gap-2">
          <span className="rounded-full bg-violet-100 px-3 py-1 text-xs font-medium text-violet-700 dark:bg-violet-900/40 dark:text-violet-300">
            {DOMAIN_LABELS[question.domain]}
          </span>
          <span className="rounded-full bg-neutral-200 px-3 py-1 text-xs font-medium text-neutral-600 dark:bg-neutral-800 dark:text-neutral-300">
            {question.type === "multi" ? "Select all that apply" : "Select one"}
          </span>
        </div>

        {/* Prompts may contain inline `code` or fenced code - render as markdown. */}
        <div className="mb-5 text-lg font-medium leading-relaxed">
          <Markdown content={question.prompt} />
        </div>

        <div className="flex flex-col gap-2">
          {orderedOptions.map((option, i) => {
            const isSelected = selected.includes(option.id);
            const isCorrectOption = feedback?.correctOptionIds.includes(option.id);
            let stateClasses =
              "border-black/10 dark:border-white/10 hover:border-violet-400";
            if (submitted) {
              if (isCorrectOption) {
                stateClasses = "border-emerald-500 bg-emerald-50 dark:bg-emerald-950/40";
              } else if (isSelected && !isCorrectOption) {
                stateClasses = "border-red-500 bg-red-50 dark:bg-red-950/40";
              }
            } else if (isSelected) {
              stateClasses = "border-violet-500 bg-violet-50 dark:bg-violet-950/30";
            }

            return (
              <button
                key={option.id}
                type="button"
                disabled={submitted}
                onClick={() => toggleOption(option.id)}
                className={`flex items-center gap-3 rounded-xl border px-4 py-3 text-left text-sm transition ${stateClasses}`}
              >
                <span className="flex h-6 w-6 flex-shrink-0 items-center justify-center rounded-full border border-current text-xs opacity-60">
                  {i + 1}
                </span>
                <span>{option.text}</span>
              </button>
            );
          })}
        </div>

        {submitted && feedback && (
          <div
            className={`mt-5 rounded-xl p-4 text-sm ${
              feedback.isCorrect
                ? "bg-emerald-100 text-emerald-800 dark:bg-emerald-950/50 dark:text-emerald-200"
                : "bg-red-100 text-red-800 dark:bg-red-950/50 dark:text-red-200"
            }`}
          >
            <p className="mb-1 font-semibold">
              {feedback.isCorrect ? "Correct!" : "Not quite."}
            </p>
            <div className="text-neutral-700 dark:text-neutral-300">
              <Markdown content={question.explanation} />
            </div>

            {aiExplainEnabled && (
              <div className="mt-3 border-t border-black/10 pt-3 dark:border-white/10">
                {aiThread.length === 0 && (
                  <button
                    type="button"
                    onClick={handleExplainFurther}
                    disabled={aiLoading}
                    className="mb-2 text-xs font-medium text-violet-700 underline decoration-dotted hover:text-violet-500 dark:text-violet-300"
                  >
                    {aiLoading ? "Thinking..." : "Explain further"}
                  </button>
                )}

                {aiThread.length > 0 && (
                  <div className="mb-3 flex flex-col gap-3">
                    {aiThread.map((m, i) => (
                      <div key={i}>
                        {m.role === "user" ? (
                          <div className="text-xs">
                            <p className="mb-1 font-medium text-neutral-500">You asked:</p>
                            <Markdown content={m.content} />
                          </div>
                        ) : (
                          <div className="text-neutral-700 dark:text-neutral-300">
                            <Markdown content={m.content} />
                          </div>
                        )}
                      </div>
                    ))}
                  </div>
                )}

                {aiLoading && <p className="mb-2 text-xs text-neutral-400">Thinking...</p>}

                <form onSubmit={handleFollowUp} className="flex items-end gap-2">
                  <MessageInput
                    value={aiFollowUp}
                    onChange={setAiFollowUp}
                    onSubmit={submitFollowUp}
                    maxRows={6}
                    placeholder={
                      aiThread.length === 0
                        ? "Or ask your own question about this... (Shift+Enter for a new line)"
                        : "Still confused? Ask a follow-up..."
                    }
                    disabled={aiLoading}
                    className="flex-1 rounded-lg border border-black/10 bg-white px-3 py-2 text-xs outline-none focus:border-violet-400 dark:border-white/10 dark:bg-neutral-950"
                  />
                  <button
                    type="submit"
                    disabled={aiLoading || !aiFollowUp.trim()}
                    className="rounded-lg bg-violet-600 px-3 py-2 text-xs font-semibold text-white transition hover:bg-violet-500 disabled:cursor-not-allowed disabled:opacity-40"
                  >
                    Ask
                  </button>
                </form>
              </div>
            )}
          </div>
        )}

        <div className="mt-6 flex justify-end">
          {!submitted ? (
            <button
              type="button"
              disabled={selected.length === 0 || pending}
              onClick={handleSubmit}
              className="rounded-lg bg-violet-600 px-5 py-2.5 text-sm font-semibold text-white transition hover:bg-violet-500 disabled:cursor-not-allowed disabled:opacity-40"
            >
              Submit
            </button>
          ) : (
            <button
              type="button"
              disabled={pending}
              onClick={handleNext}
              className="rounded-lg bg-violet-600 px-5 py-2.5 text-sm font-semibold text-white transition hover:bg-violet-500"
            >
              {index + 1 >= total ? "Finish" : "Next"}
            </button>
          )}
        </div>
      </div>

      <p className="mt-4 text-center text-xs text-neutral-400">
        Tip: press number keys to select an answer, Enter to submit / continue.
      </p>
    </div>
  );
}
```

> The original rendered the prompt and explanation as plain `<p>`. This template renders them through
> `<Markdown>` so code-heavy subjects can use backticks/fences in questions and explanations. For
> non-technical subjects it is harmless. The `Markdown` explanation wrapper inherits the banner colour,
> which is the intended look.

## `src/app/results/[attemptId]/page.tsx`

```tsx
import Link from "next/link";
import { notFound, redirect } from "next/navigation";
import { prisma } from "@/lib/prisma";
import { DOMAIN_LABELS, Domain } from "@/data/types";

export const dynamic = "force-dynamic";

export default async function ResultsPage({
  params,
}: {
  params: Promise<{ attemptId: string }>;
}) {
  const { attemptId } = await params;
  const attempt = await prisma.attempt.findUnique({
    where: { id: attemptId },
    include: { answers: true },
  });

  if (!attempt) notFound();
  if (!attempt.finishedAt) redirect(`/quiz/${attempt.id}`);

  const breakdown = new Map<Domain, { correct: number; total: number }>();
  for (const answer of attempt.answers) {
    const domain = answer.domain as Domain;
    const entry = breakdown.get(domain) ?? { correct: 0, total: 0 };
    entry.total += 1;
    if (answer.isCorrect) entry.correct += 1;
    breakdown.set(domain, entry);
  }

  const rows = Array.from(breakdown.entries())
    .map(([domain, stats]) => ({
      domain,
      ...stats,
      pct: stats.total > 0 ? Math.round((stats.correct / stats.total) * 100) : 0,
    }))
    .sort((a, b) => a.pct - b.pct);

  const scorePct = Math.round(attempt.scorePct ?? 0);

  return (
    <div className="mx-auto max-w-2xl px-4 py-12 sm:px-6">
      <div className="mb-8 rounded-2xl border border-black/10 bg-neutral-50 p-8 text-center dark:border-white/10 dark:bg-neutral-900">
        <p className="text-sm font-medium uppercase tracking-wide text-neutral-500">Your score</p>
        <p
          className={`mt-2 text-5xl font-bold ${
            scorePct >= 80
              ? "text-emerald-600 dark:text-emerald-400"
              : scorePct >= 60
                ? "text-amber-500"
                : "text-red-500"
          }`}
        >
          {scorePct}%
        </p>
        <p className="mt-2 text-neutral-600 dark:text-neutral-400">
          {attempt.correctCount} / {attempt.totalQuestions} correct
        </p>
      </div>

      <h2 className="mb-3 text-lg font-semibold">Breakdown by domain</h2>
      <div className="overflow-hidden rounded-xl border border-black/10 dark:border-white/10">
        <table className="w-full text-sm">
          <thead className="bg-neutral-100 text-left text-neutral-500 dark:bg-neutral-900">
            <tr>
              <th className="px-4 py-3 font-medium">Domain</th>
              <th className="px-4 py-3 font-medium">Correct</th>
              <th className="px-4 py-3 font-medium">Score</th>
            </tr>
          </thead>
          <tbody>
            {rows.map((row) => (
              <tr key={row.domain} className="border-t border-black/10 dark:border-white/10">
                <td className="px-4 py-3">{DOMAIN_LABELS[row.domain] ?? row.domain}</td>
                <td className="px-4 py-3 text-neutral-500">
                  {row.correct}/{row.total}
                </td>
                <td
                  className={`px-4 py-3 font-medium ${
                    row.pct < 60
                      ? "text-red-500"
                      : row.pct < 80
                        ? "text-amber-500"
                        : "text-emerald-600 dark:text-emerald-400"
                  }`}
                >
                  {row.pct}%
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      <div className="mt-8 flex gap-3">
        <Link
          href="/"
          className="rounded-lg bg-violet-600 px-5 py-2.5 text-sm font-semibold text-white transition hover:bg-violet-500"
        >
          New quiz
        </Link>
        <Link
          href="/history"
          className="rounded-lg border border-black/15 px-5 py-2.5 text-sm font-semibold transition hover:bg-black/5 dark:border-white/15 dark:hover:bg-white/5"
        >
          View history
        </Link>
      </div>
    </div>
  );
}
```

## `src/app/history/page.tsx`

```tsx
import Link from "next/link";
import { prisma } from "@/lib/prisma";
import { DOMAIN_LABELS } from "@/data/types";
import { getWeakAreas } from "@/lib/stats";

export const dynamic = "force-dynamic";

export default async function HistoryPage() {
  const attempts = await prisma.attempt.findMany({
    where: { finishedAt: { not: null } },
    orderBy: { startedAt: "desc" },
    take: 50,
  });

  const weakAreas = await getWeakAreas();

  return (
    <div className="mx-auto max-w-3xl px-4 py-12 sm:px-6">
      <h1 className="mb-8 text-3xl font-bold tracking-tight">History &amp; Weak Areas</h1>

      <section className="mb-10">
        <h2 className="mb-3 text-lg font-semibold">Weak areas (all attempts combined)</h2>
        {weakAreas.length === 0 ? (
          <p className="text-neutral-500">Finish a quiz to start building this up.</p>
        ) : (
          <div className="overflow-hidden rounded-xl border border-black/10 dark:border-white/10">
            <table className="w-full text-sm">
              <thead className="bg-neutral-100 text-left text-neutral-500 dark:bg-neutral-900">
                <tr>
                  <th className="px-4 py-3 font-medium">Domain</th>
                  <th className="px-4 py-3 font-medium">Answered</th>
                  <th className="px-4 py-3 font-medium">Accuracy</th>
                </tr>
              </thead>
              <tbody>
                {weakAreas.map((row) => (
                  <tr key={row.domain} className="border-t border-black/10 dark:border-white/10">
                    <td className="px-4 py-3">{DOMAIN_LABELS[row.domain] ?? row.domain}</td>
                    <td className="px-4 py-3 text-neutral-500">
                      {row.correct}/{row.total}
                    </td>
                    <td className="px-4 py-3">
                      <div className="flex items-center gap-2">
                        <div className="h-2 w-24 overflow-hidden rounded-full bg-neutral-200 dark:bg-neutral-800">
                          <div
                            className={`h-full rounded-full ${
                              row.pct < 60
                                ? "bg-red-500"
                                : row.pct < 80
                                  ? "bg-amber-500"
                                  : "bg-emerald-500"
                            }`}
                            style={{ width: `${row.pct}%` }}
                          />
                        </div>
                        <span
                          className={
                            row.pct < 60
                              ? "text-red-500"
                              : row.pct < 80
                                ? "text-amber-500"
                                : "text-emerald-600 dark:text-emerald-400"
                          }
                        >
                          {row.pct}%
                        </span>
                      </div>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </section>

      <section>
        <h2 className="mb-3 text-lg font-semibold">Past attempts</h2>
        {attempts.length === 0 ? (
          <p className="text-neutral-500">No completed attempts yet.</p>
        ) : (
          <div className="flex flex-col gap-2">
            {attempts.map((attempt) => (
              <Link
                key={attempt.id}
                href={`/results/${attempt.id}`}
                className="flex items-center justify-between rounded-xl border border-black/10 px-4 py-3 text-sm transition hover:border-violet-400 dark:border-white/10"
              >
                <span className="text-neutral-500">
                  {attempt.startedAt.toLocaleString()} &middot; {attempt.totalQuestions} questions
                </span>
                <span
                  className={`font-semibold ${
                    (attempt.scorePct ?? 0) >= 80
                      ? "text-emerald-600 dark:text-emerald-400"
                      : (attempt.scorePct ?? 0) >= 60
                        ? "text-amber-500"
                        : "text-red-500"
                  }`}
                >
                  {Math.round(attempt.scorePct ?? 0)}%
                </span>
              </Link>
            ))}
          </div>
        )}
      </section>
    </div>
  );
}
```

(`DOMAIN_LABELS` import in history is used; keep it.)

---

## Chat

### `src/app/chat/layout.tsx`

```tsx
import Link from "next/link";
import { listConversations } from "@/lib/chatActions";
import { ConversationList } from "./ConversationList";

export const dynamic = "force-dynamic";

export default async function ChatLayout({ children }: { children: React.ReactNode }) {
  const conversations = await listConversations();

  return (
    <div className="flex h-[calc(100dvh-4rem)]">
      <aside className="hidden w-64 shrink-0 flex-col border-r border-black/10 md:flex dark:border-white/10">
        <div className="p-3">
          <Link
            href="/chat"
            className="block rounded-lg bg-violet-600 px-3 py-2 text-center text-sm font-semibold text-white transition hover:bg-violet-500"
          >
            + New chat
          </Link>
        </div>
        <div className="flex-1 overflow-y-auto px-2 pb-3">
          <ConversationList conversations={conversations} />
        </div>
      </aside>
      <div className="flex min-h-0 flex-1 flex-col">{children}</div>
    </div>
  );
}
```

### `src/app/chat/ConversationList.tsx`

```tsx
"use client";

import Link from "next/link";
import { usePathname, useRouter } from "next/navigation";
import { deleteConversation } from "@/lib/chatActions";

interface ConversationSummary {
  id: string;
  title: string;
  updatedAt: Date;
}

export function ConversationList({ conversations }: { conversations: ConversationSummary[] }) {
  const pathname = usePathname();
  const router = useRouter();

  async function handleDelete(e: React.MouseEvent, id: string) {
    e.preventDefault();
    e.stopPropagation();
    await deleteConversation(id);
    if (pathname === `/chat/${id}`) {
      router.push("/chat");
    } else {
      router.refresh();
    }
  }

  if (conversations.length === 0) {
    return <p className="px-3 py-2 text-xs text-neutral-400">No saved chats yet.</p>;
  }

  return (
    <div className="flex flex-col gap-0.5">
      {conversations.map((c) => {
        const href = `/chat/${c.id}`;
        const active = pathname === href;
        return (
          <Link
            key={c.id}
            href={href}
            className={`group flex items-center justify-between gap-2 rounded-lg px-3 py-2 text-sm transition ${
              active
                ? "bg-violet-100 text-violet-900 dark:bg-violet-900/40 dark:text-violet-100"
                : "text-neutral-600 hover:bg-black/5 dark:text-neutral-300 dark:hover:bg-white/5"
            }`}
          >
            <span className="truncate">{c.title}</span>
            <button
              type="button"
              onClick={(e) => handleDelete(e, c.id)}
              aria-label="Delete chat"
              className="shrink-0 rounded px-1 text-xs text-neutral-400 opacity-0 transition hover:text-red-500 group-hover:opacity-100"
            >
              ✕
            </button>
          </Link>
        );
      })}
    </div>
  );
}
```

### `src/app/chat/page.tsx`

```tsx
import { getWeakAreas, meaningfulWeakDomains, weakAreaSummaryText } from "@/lib/stats";
import { pickExamplePrompts } from "@/data/examplePrompts";
import { ChatPanel } from "./ChatPanel";

export const dynamic = "force-dynamic";

export default async function ChatPage() {
  const aiEnabled = Boolean(process.env.OPENAI_API_KEY);

  if (!aiEnabled) {
    return (
      <div className="mx-auto flex h-full max-w-2xl items-center px-4 sm:px-6">
        <div className="rounded-xl border border-amber-500/30 bg-amber-50 p-4 text-sm text-amber-800 dark:bg-amber-950/30 dark:text-amber-200">
          Set <code>OPENAI_API_KEY</code> in <code>.env.local</code> to enable this chat.
        </div>
      </div>
    );
  }

  const weakAreas = await getWeakAreas();
  const weakAreaSummary = weakAreaSummaryText(weakAreas);
  const examplePrompts = pickExamplePrompts(meaningfulWeakDomains(weakAreas));

  return (
    <ChatPanel
      key="new-chat"
      conversationId={null}
      initialMessages={[]}
      weakAreaSummary={weakAreaSummary}
      examplePrompts={examplePrompts}
    />
  );
}
```

### `src/app/chat/[id]/page.tsx`

```tsx
import { notFound } from "next/navigation";
import { getConversation } from "@/lib/chatActions";
import { getWeakAreas, meaningfulWeakDomains, weakAreaSummaryText } from "@/lib/stats";
import { pickExamplePrompts } from "@/data/examplePrompts";
import { ChatPanel } from "../ChatPanel";

export const dynamic = "force-dynamic";

export default async function ChatConversationPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const conversation = await getConversation(id);
  if (!conversation) notFound();

  const weakAreas = await getWeakAreas();
  const weakAreaSummary = weakAreaSummaryText(weakAreas);
  const examplePrompts = pickExamplePrompts(meaningfulWeakDomains(weakAreas));

  return (
    <ChatPanel
      key={conversation.id}
      conversationId={conversation.id}
      initialMessages={conversation.messages.map((m) => ({
        role: m.role as "user" | "assistant",
        content: m.content,
        truncated: m.truncated,
      }))}
      weakAreaSummary={weakAreaSummary}
      examplePrompts={examplePrompts}
    />
  );
}
```

### `src/app/chat/ChatPanel.tsx`

See `ai-integration.md` for the full file (it is tightly coupled to the API contract and truncation logic).

### `src/app/api/ai/route.ts`

See `ai-integration.md`.

---

## `README.md` (generate, adapt wording)

Must cover: what the app is; run steps (`npm install`, `npx prisma migrate dev`, `npm run dev`);
folder map (questions, quiz.ts, actions.ts, schema, QuizRunner); how to add questions (id unique,
domain, type, options with `correct`, explanation); optional AI setup (`OPENAI_API_KEY`,
`OPENAI_MODEL`); notes (no auth, single-user local, `prisma/dev.db` gitignored, content is original).

## Commands cheat-sheet

```bash
npx prisma migrate dev --name init     # first run
npx prisma studio                      # inspect history (optional)
npx tsc --noEmit && npm run lint && npm run build
npm run dev                            # http://localhost:3000
```
