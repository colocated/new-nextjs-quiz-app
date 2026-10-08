# AI integration (OpenAI)

One proxy endpoint, three consumers. No SDK, no streaming - a plain `fetch` to Chat Completions.

```
QuizRunner "Explain further" + follow-ups ─┐
ChatPanel (/chat, /chat/[id])  ────────────┼──> POST /api/ai ──> api.openai.com/v1/chat/completions
Continue button (re-sends history) ────────┘
```

The **caller** builds the system prompt and history; the route only validates and proxies. This
keeps the route tiny and lets each surface give the model different context.

## Feature gating

AI is optional. `process.env.OPENAI_API_KEY` unset ->
- Route returns **501** `{ error: "AI features are not configured." }`
- Quiz page passes `aiExplainEnabled={false}` -> the "Explain further" block is not rendered
- `/chat` shows the amber setup hint instead of the chat UI
- Nothing else calls the network.

Env vars (`.env.local`): `OPENAI_API_KEY`, `OPENAI_MODEL` (default `gpt-5.4-mini`; the original also
honoured legacy `OPENAI_EXPLAIN_MODEL`). To use another model, just set `OPENAI_MODEL`. The route
deliberately sends no `temperature` so it works with reasoning models; if you switch to an older
non-reasoning model and want a custom temperature, add it back.

## `src/app/api/ai/route.ts`

```ts
import { NextResponse } from "next/server";

interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

const MAX_MESSAGES = 40;
const MAX_MESSAGE_LENGTH = 4000;
// Assistant replies can grow past the user limit once stitched together via
// the chat's Continue button, and they get sent back as history.
const MAX_ASSISTANT_MESSAGE_LENGTH = 20000;

function isValidMessages(value: unknown): value is ChatMessage[] {
  if (!Array.isArray(value) || value.length === 0 || value.length > MAX_MESSAGES) return false;
  return value.every(
    (m) =>
      m &&
      typeof m === "object" &&
      (m.role === "system" || m.role === "user" || m.role === "assistant") &&
      typeof m.content === "string" &&
      m.content.length > 0 &&
      m.content.length <= (m.role === "assistant" ? MAX_ASSISTANT_MESSAGE_LENGTH : MAX_MESSAGE_LENGTH)
  );
}

// Single AI endpoint shared by: the quiz's "explain further" thread, its
// follow-up questions, and the standalone /chat page. The caller builds
// whatever system prompt + history fits its context; this route just proxies
// the conversation to the configured model.
export async function POST(request: Request) {
  const apiKey = process.env.OPENAI_API_KEY;
  if (!apiKey) {
    return NextResponse.json({ error: "AI features are not configured." }, { status: 501 });
  }

  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ error: "Invalid JSON body." }, { status: 400 });
  }

  const messages = (body as { messages?: unknown })?.messages;
  if (!isValidMessages(messages)) {
    return NextResponse.json({ error: "Missing or invalid `messages` array." }, { status: 400 });
  }

  try {
    const res = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify({
        model: process.env.OPENAI_MODEL || process.env.OPENAI_EXPLAIN_MODEL || "gpt-5.4-mini",
        messages,
        // No `temperature`: GPT-5-family reasoning models may reject non-default values.
        // Reasoning tokens count toward this cap, so leave headroom above the visible answer.
        max_completion_tokens: 1200,
      }),
    });

    if (!res.ok) {
      const text = await res.text();
      return NextResponse.json({ error: `OpenAI error: ${text}` }, { status: 502 });
    }

    const data = await res.json();
    const choice = data.choices?.[0];
    const content: string | undefined = choice?.message?.content;
    return NextResponse.json({
      content: content ?? "No response returned.",
      // "length" means the model hit max_completion_tokens mid-answer.
      truncated: choice?.finish_reason === "length",
    });
  } catch {
    return NextResponse.json({ error: "Failed to reach OpenAI." }, { status: 502 });
  }
}
```

Hardening notes (keep as-is unless asked): the key never reaches the browser (server env only);
payload caps bound cost; error bodies are passed through to the client for a single-user local tool -
if the app is ever deployed publicly, replace the raw OpenAI error text with a generic message, add
auth/rate-limiting, and don't expose `/api/ai` unauthenticated.

## System prompts

Build these from `APP_CONFIG` + the active domain. Pattern, from the original app:

**Per-question tutor** (in `QuizRunner`, rebuilt for every question, joined with `\n`):
```
You are <tutorRole> helping a student who just answered a practice question about <subject>.
Learner context: <learnerContext>                       (only if non-empty)
Domain: <domain label>
Question: <prompt>
Reference explanation: <explanation>
Give short, practical answers (a few sentences, or a brief code block in a ```<codeFence> fence when a sample genuinely helps). Stay focused on this question and concepts related to it. If the student asks a follow-up, address it directly rather than repeating the whole explanation.
```
"Explain further" sends the user message `"Give a deeper, more practical explanation of this question and its answer."` with an empty thread. Follow-ups send `[system, ...thread, userFollowUp]`. The thread is per-question and cleared on Next.

**Standalone chat tutor** (in `ChatPanel`):
```
You are a helpful, precise <expert role> assisting someone studying <subject / goal>.
Use ```<codeFence> code fences for any code samples. Be concise but thorough enough to actually teach the concept. Prefer accurate, idiomatic <subject> over verbosity.
Context: this student's weakest areas so far, based on their practice quiz history, are: <weakAreaSummary>. Lean on that context if it's relevant to what they ask, but don't force it into unrelated answers.      (only if weakAreaSummary non-empty)
Learner context: <learnerContext>                      (only if non-empty; no personal data)
```

When the learner was personalised from a CV, `learnerContext` should be a skills-level sentence like
"Mid-level backend engineer moving toward platform/SRE roles; strong in Java, weaker in Kubernetes
and networking." - never names, employers, or contact details.

## Truncation & Continue (important UX detail)

`max_completion_tokens: 1200` means long answers get cut. The route returns `truncated: true` when
`finish_reason === "length"`. The chat then shows a pill **"Response was cut off · Continue"**.
Continue sends the full history plus a hidden user instruction, and *stitches* the result onto the
previous assistant message (the instruction is never shown or saved):

- If the cut-off happened **inside an open code fence** (odd number of ```` ``` ```` lines), the
  instruction adds: don't start a new fence; write the closing fence only when the code is finished.
- `cleanContinuation` strips a leading re-opened fence if the model ignores that.
- `truncated` is persisted on `ChatMessage` so reloading a conversation still offers Continue; a
  fence-parity check also covers older rows.

## `src/app/chat/ChatPanel.tsx` (full file)

```tsx
"use client";

import { FormEvent, useEffect, useRef, useState } from "react";
import { useRouter } from "next/navigation";
import { Markdown } from "@/components/Markdown";
import { MessageInput } from "@/components/MessageInput";
import { APP_CONFIG } from "@/data/app";
import { DOMAIN_LABELS } from "@/data/types";
import { ExamplePrompt } from "@/data/examplePrompts";
import {
  appendMessage,
  createConversationWithMessage,
  extendLastAssistantMessage,
} from "@/lib/chatActions";

interface ChatMessage {
  role: "user" | "assistant";
  content: string;
  // Saved from the API's finish_reason; only present on reloaded messages.
  truncated?: boolean;
}

const REQUEST_TIMEOUT_MS = 45_000;

function hasOpenCodeFence(text: string) {
  return (text.match(/^\s*```/gm)?.length ?? 0) % 2 === 1;
}

function continuePrompt(previous: string) {
  return [
    "Your previous reply was cut off. Continue exactly where you left off, even mid-sentence or mid-line. Do not repeat anything already written and do not add any preamble.",
    hasOpenCodeFence(previous)
      ? "The cut-off happened INSIDE an open code block, so output the remaining raw code directly. Do NOT start a new code fence; only write a closing fence once the code is finished."
      : "",
  ]
    .filter(Boolean)
    .join(" ");
}

// Safety net for when the model re-opens a code fence anyway: drop it, since
// the original fence is still open and will be closed by the continuation.
function cleanContinuation(previous: string, continuation: string) {
  if (!hasOpenCodeFence(previous)) return continuation;
  return continuation.replace(/^\s*```[\w-]*[ \t]*\r?\n/, "");
}

function systemPrompt(weakAreaSummary: string) {
  return [
    `You are a helpful, precise expert assisting someone studying ${APP_CONFIG.subject}. You are ${APP_CONFIG.tutorRole}.`,
    `Use \`\`\`${APP_CONFIG.codeFence} code fences for any code samples. Be concise but thorough enough to actually teach the concept. Prefer accurate, idiomatic answers over verbosity.`,
    APP_CONFIG.learnerContext ? `Learner context: ${APP_CONFIG.learnerContext}` : "",
    weakAreaSummary
      ? `Context: this student's weakest areas so far, based on their practice quiz history, are: ${weakAreaSummary}. Lean on that context if it's relevant to what they ask, but don't force it into unrelated answers.`
      : "",
  ]
    .filter(Boolean)
    .join("\n");
}

function endsInsideCodeFence(messages: ChatMessage[]) {
  const last = messages[messages.length - 1];
  // The saved finish_reason is authoritative; the fence check also catches
  // replies saved before that flag existed.
  return last?.role === "assistant" && (Boolean(last.truncated) || hasOpenCodeFence(last.content));
}

function SendIcon() {
  return (
    <svg viewBox="0 0 20 20" fill="currentColor" className="h-4 w-4">
      <path d="M3.4 2.5a.75.75 0 0 1 .8-.13l13 5.5a.75.75 0 0 1 0 1.38l-13 5.5a.75.75 0 0 1-1.05-.85l1.6-5.15-1.6-5.15a.75.75 0 0 1 .25-1.1Z" />
    </svg>
  );
}

function Spinner() {
  return (
    <svg viewBox="0 0 24 24" fill="none" className="h-4 w-4 animate-spin">
      <circle cx="12" cy="12" r="9" stroke="currentColor" strokeOpacity="0.25" strokeWidth="3" />
      <path d="M21 12a9 9 0 0 0-9-9" stroke="currentColor" strokeWidth="3" strokeLinecap="round" />
    </svg>
  );
}

interface ChatPanelProps {
  conversationId: string | null;
  initialMessages: ChatMessage[];
  weakAreaSummary: string;
  examplePrompts: ExamplePrompt[];
}

export function ChatPanel({
  conversationId: initialConversationId,
  initialMessages,
  weakAreaSummary,
  examplePrompts,
}: ChatPanelProps) {
  const router = useRouter();
  const [conversationId, setConversationId] = useState(initialConversationId);
  const [messages, setMessages] = useState<ChatMessage[]>(initialMessages);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  // Whether the latest assistant reply was cut off (token cap or unclosed code fence).
  const [truncated, setTruncated] = useState(() => endsInsideCodeFence(initialMessages));
  const bottomRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLTextAreaElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, loading]);

  async function send(rawContent: string) {
    const content = rawContent.trim();
    if (!content || loading) return;

    setInput("");
    setError(null);
    setTruncated(false);
    const next = [...messages, { role: "user" as const, content }];
    setMessages(next);
    setLoading(true);

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MS);

    const isNewConversation = !conversationId;

    try {
      // Persist the user's message first (creating the conversation on first
      // send) so it's saved even if the AI call below fails or times out.
      let activeConversationId = conversationId;
      if (activeConversationId) {
        await appendMessage(activeConversationId, "user", content);
      } else {
        activeConversationId = await createConversationWithMessage(content);
        setConversationId(activeConversationId);
        // Don't navigate to /chat/[id] yet - that swaps in a different page
        // component and remounts this panel. Wait until the full exchange
        // (including the assistant's reply) is saved, so the remount never
        // has a chance to drop an in-flight response.
      }

      const res = await fetch("/api/ai", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        signal: controller.signal,
        body: JSON.stringify({
          messages: [{ role: "system", content: systemPrompt(weakAreaSummary) }, ...next],
        }),
      });
      const data = await res.json();
      if (!res.ok) {
        setError(data.error ?? "Something went wrong.");
        setMessages(messages);
        setInput(content);
        return;
      }

      setMessages([...next, { role: "assistant", content: data.content }]);
      setTruncated(Boolean(data.truncated));
      await appendMessage(activeConversationId, "assistant", data.content, Boolean(data.truncated));

      if (isNewConversation) {
        router.replace(`/chat/${activeConversationId}`, { scroll: false });
      }
    } catch (err) {
      setError(
        err instanceof DOMException && err.name === "AbortError"
          ? "The request timed out. Try again."
          : "Failed to reach the AI chat."
      );
      setMessages(messages);
      setInput(content);
    } finally {
      clearTimeout(timeout);
      setLoading(false);
      inputRef.current?.focus();
    }
  }

  async function continueReply() {
    const last = messages[messages.length - 1];
    if (!conversationId || last?.role !== "assistant" || loading) return;

    setError(null);
    setTruncated(false);
    setLoading(true);

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MS);

    try {
      // The continue instruction is sent to the model but never shown or saved;
      // the result is stitched onto the existing assistant message instead.
      const res = await fetch("/api/ai", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        signal: controller.signal,
        body: JSON.stringify({
          messages: [
            { role: "system", content: systemPrompt(weakAreaSummary) },
            ...messages,
            { role: "user", content: continuePrompt(last.content) },
          ],
        }),
      });
      const data = await res.json();
      if (!res.ok) {
        setError(data.error ?? "Something went wrong.");
        setTruncated(true);
        return;
      }

      const extra = cleanContinuation(last.content, data.content);
      setMessages([...messages.slice(0, -1), { role: "assistant", content: last.content + extra }]);
      setTruncated(Boolean(data.truncated));
      await extendLastAssistantMessage(conversationId, extra, Boolean(data.truncated));
    } catch (err) {
      setError(
        err instanceof DOMException && err.name === "AbortError"
          ? "The request timed out. Try again."
          : "Failed to reach the AI chat."
      );
      setTruncated(true);
    } finally {
      clearTimeout(timeout);
      setLoading(false);
    }
  }

  function handleSubmit(e: FormEvent) {
    e.preventDefault();
    send(input);
  }

  function handleEnterSubmit() {
    send(input);
  }

  return (
    <div className="flex min-h-0 flex-1 flex-col">
      <div className="flex shrink-0 items-center justify-between border-b border-black/10 px-4 py-3 sm:px-6 dark:border-white/10">
        <div>
          <h1 className="text-base font-semibold">Ask about {APP_CONFIG.askAiNoun}</h1>
          <p className="text-xs text-neutral-500">
            Separate from the quiz &middot; saved automatically as you go
          </p>
        </div>
      </div>

      <div className="min-h-0 flex-1 overflow-y-auto">
        {messages.length === 0 ? (
          <div className="flex h-full items-center justify-center p-6">
            <div className="w-full max-w-xl text-center">
              <p className="mb-5 text-sm text-neutral-400">
                Ask anything about {APP_CONFIG.askAiNoun}, or start from one of these:
              </p>
              <div className="grid grid-cols-1 gap-3 sm:grid-cols-2">
                {examplePrompts.map((ex, i) => (
                  <button
                    key={i}
                    type="button"
                    onClick={() => send(ex.prompt)}
                    disabled={loading}
                    className="flex h-full flex-col gap-1.5 rounded-xl border border-black/10 bg-white p-3.5 text-left text-sm transition hover:border-violet-400 hover:shadow-sm disabled:cursor-not-allowed disabled:opacity-40 dark:border-white/10 dark:bg-neutral-950"
                  >
                    <span className="w-fit rounded-full bg-violet-100 px-2 py-0.5 text-[10px] font-medium text-violet-700 dark:bg-violet-900/40 dark:text-violet-300">
                      {DOMAIN_LABELS[ex.domain]}
                    </span>
                    <span className="text-neutral-700 dark:text-neutral-300">{ex.prompt}</span>
                  </button>
                ))}
              </div>
            </div>
          </div>
        ) : (
          <div className="mx-auto max-w-3xl px-4 py-6 sm:px-6">
            <div className="flex flex-col gap-4">
              {messages.map((m, i) => (
                <div
                  key={i}
                  className={
                    m.role === "user"
                      ? "ml-auto max-w-[85%] rounded-2xl rounded-br-sm bg-violet-600 px-4 py-2.5 text-sm text-white"
                      : "mr-auto max-w-[85%] rounded-2xl rounded-bl-sm bg-neutral-100 px-4 py-2.5 text-sm dark:bg-neutral-900"
                  }
                >
                  <Markdown content={m.content} tone={m.role === "user" ? "inverted" : "default"} />
                </div>
              ))}
              {truncated && !loading && (
                <button
                  type="button"
                  onClick={continueReply}
                  className="mr-auto rounded-full border border-violet-400 px-3.5 py-1.5 text-xs font-medium text-violet-700 transition hover:bg-violet-50 dark:text-violet-300 dark:hover:bg-violet-950/40"
                >
                  Response was cut off &middot; Continue
                </button>
              )}
              {loading && (
                <div className="mr-auto flex items-center gap-2 rounded-2xl rounded-bl-sm bg-neutral-100 px-4 py-2.5 text-xs text-neutral-500 dark:bg-neutral-900 dark:text-neutral-400">
                  <Spinner />
                  Thinking...
                </div>
              )}
              {error && (
                <p className="mr-auto max-w-[85%] rounded-2xl rounded-bl-sm bg-red-100 px-4 py-2.5 text-xs text-red-700 dark:bg-red-950/50 dark:text-red-300">
                  {error}
                </p>
              )}
            </div>
            <div ref={bottomRef} />
          </div>
        )}
      </div>

      <div className="shrink-0 border-t border-black/10 px-4 py-3 sm:px-6 dark:border-white/10">
        <form onSubmit={handleSubmit} className="mx-auto flex max-w-3xl items-end gap-2">
          <MessageInput
            ref={inputRef}
            value={input}
            onChange={setInput}
            onSubmit={handleEnterSubmit}
            placeholder={`Ask a ${APP_CONFIG.askAiNoun} question... (Shift+Enter for a new line)`}
            autoFocus
            className="flex-1 rounded-lg border border-black/10 bg-white px-3 py-2.5 text-sm outline-none focus:border-violet-400 dark:border-white/10 dark:bg-neutral-950"
          />
          <button
            type="submit"
            disabled={loading || !input.trim()}
            className="flex items-center gap-2 rounded-lg bg-violet-600 px-4 py-2.5 text-sm font-semibold text-white transition hover:bg-violet-500 disabled:cursor-not-allowed disabled:opacity-40"
          >
            {loading ? <Spinner /> : <SendIcon />}
            {loading ? "Sending" : "Send"}
          </button>
        </form>
      </div>
    </div>
  );
}
```

Key behaviours baked into that file (preserve if you refactor):
1. The user's message is persisted **before** the AI call (survives failures/timeouts).
2. New conversations are created on first send but the route only changes to `/chat/<id>` (via `router.replace`, `scroll:false`) **after** the assistant reply is saved, to avoid a remount dropping the in-flight response.
3. On failure the optimistic message is rolled back and the text restored to the input.
4. 45 s client timeout via `AbortController`.
5. Auto-scroll to the bottom on new messages/loading.
6. Conversation title = first user message, whitespace-collapsed, truncated to 60 chars (`57 + "..."`).

## Testing the AI without a key

Without a key you can still verify: `/chat` shows the hint; `POST /api/ai` returns 501;
the quiz hides "Explain further". If the user supplies a key, smoke-test with one real call
(`curl -X POST localhost:3000/api/ai -H 'content-type: application/json' -d '{"messages":[{"role":"user","content":"hi"}]}'`)
and expect `{content, truncated:false}`.
