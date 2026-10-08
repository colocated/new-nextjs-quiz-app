---
name: new-nextjs-quiz-app
description: Scaffold a complete, personalised Next.js quiz/trainer app from scratch in an empty directory - randomized multi-domain quizzes, score history and weak-area tracking (Prisma + SQLite), an OpenAI-powered "Explain further" tutor and standalone Ask-AI chat, violet/neutral Tailwind design system with dark mode. The model writes every file, the whole question bank, and the AI prompts itself. Accepts a title, an optional description, and optionally an attached CV/resume/syllabus to personalise the content. Use when the user runs /new-nextjs-quiz-app or asks to build a study/quiz/certification-trainer app.
argument-hint: '"<app title, e.g. Java Engineering Course>" "<optional: what I already know / what I want to target>"'
disable-model-invocation: false
---

# /new-nextjs-quiz-app

Build a full, working study-quiz web app from nothing. You (the model) design the
subject domains, write every question, write every source file, install, migrate, and
verify it runs. There is no reference code to copy from - everything you need is in this
skill folder:

| File | What it contains | When to read |
|---|---|---|
| `SKILL.md` (this file) | The end-to-end procedure, intake questions, quality bars | Now |
| `reference/architecture.md` | Stack, folder layout, data model, behaviours, every route | Before scaffolding |
| `reference/design-system.md` | Exact tokens, classes, component recipes, dark mode | Before writing any UI |
| `reference/code-templates.md` | Complete source for every non-content file | While writing code |
| `reference/question-bank.md` | Question schema, types, quality bar, authoring workflow, personalisation | Before writing questions |
| `reference/ai-integration.md` | `/api/ai` route, prompts, chat, truncation/continue, safety | Before writing AI features |
| `reference/openai-key-setup.md` | Restricted-key permissions + best practice | When writing the README and hand-off |

Read each reference file in full when its row says so. Do not skim - the details (e.g. the
`LayoutProps<"/">` type, `params` being a Promise, `force-dynamic`) are what make the app work.

## Arguments

`$ARGUMENTS` is up to two quoted strings:

1. **Title** (required) - e.g. `"Java Engineering Course"`. Becomes the app name, nav label, `<title>`, folder slug and AI persona subject.
2. **Description** (optional) - what the learner already knows or wants to target, e.g. `"I know Python well, new to the JVM; targeting backend interviews"`.

If no title was given but a file is attached, derive a title from the file (see "Attachments").
If neither exists, ask for a title and stop.

## Step 0 - Rules for this skill

- **Next.js here may differ from your training data.** After `create-next-app`, read the
  relevant guides in `node_modules/next/dist/docs/` (routing, server actions, layouts, fonts,
  `LayoutProps`, async `params`) and obey any `AGENTS.md` / `CLAUDE.md` the generator writes.
  If the installed Next version disagrees with the templates in this skill, follow the
  installed version's docs and keep the same *behaviour* and *design*.
- Work in a **fresh directory**. Default: `./<kebab-title>` under the current working
  directory. If cwd is already an empty directory, or the user says "here", use cwd. Never
  overwrite a non-empty directory - pick a new name or ask.
- Never write real secrets. Create `.env.local` with a placeholder `OPENAI_API_KEY=` for the
  user to fill in; `.env*` stays git-ignored.
- Do not `git commit`/push unless asked.
- Content must be **original** - written to teach the concepts, not transcribed from a
  vendor's proprietary exam bank or a paid course.

## Step 1 - Understand the learner (attachments + intake)

### 1a. Attachments

If the user attached or referenced files (CV/resume PDF or docx, LinkedIn export, job spec,
syllabus, certification exam guide, notes, a GitHub README), **read them all first**
(Read tool handles PDFs/images; for docx use the docx skill or unzip the XML). Extract:

- Current/past roles, seniority, years of experience
- Tech stack and tools actually used, domains of strength
- Gaps and adjacent areas (things on the CV's edges, or in a target job spec but not on the CV)
- Target direction: explicit ("looking for platform roles") or implied (recent courses, side projects, job-spec attached)
- Certifications held / in progress

State back a **3-5 line learner profile** ("You're a mid-level backend dev, 5 yrs Java/Spring,
light on Kubernetes and distributed systems; likely targeting senior/staff backend or platform roles").

**Privacy:** the CV is used only to shape content. Never paste personal details (name,
employer, contact info) into the app source, question text, README or AI system prompts.
Refer to skills and roles only.

### 1b. Establishment questions

Ask **3-6 short questions** with `AskUserQuestion` (batch them in one call, options with a
recommended default first). Skip any question the title/description/attachment already answers.
Choose from:

1. **Goal**: certification exam / job interviews / on-the-job upskilling / general mastery.
2. **Target roles/professions** (multiSelect; pre-fill options from the CV): the set of jobs the
   question bank should serve. *This is the key personalisation input.*
3. **Current level**: beginner / intermediate / advanced (calibrates difficulty mix).
4. **Focus**: breadth across everything vs. deep on weak areas (affects domain weights).
5. **Question bank size**: Standard (~120-150) / Large (~250-300) / Maximum (as many as you can
   write well, 400+) - default **Large**. If the user said "as many as you can", use Maximum.
6. **AI tutor**: wire up OpenAI now (needs `OPENAI_API_KEY` later) - default yes.
7. **Code-heavy?** whether questions should include code snippets/commands (default: yes for
   technical subjects, no otherwise). Also ask for the code language(s) to syntax-highlight.

Do not ask more than 6. If the description is rich enough, ask only 1-3 and move on.

### 1c. Write the plan

Produce a short plan *in chat* (not a file) before coding:

- App name, slug, one-line tagline
- **Domains**: 6-12 domains (see `reference/question-bank.md` "Choosing domains") with keys, labels, target question counts and the professions each serves
- Question mix target (see below) and total target
- Any personalisation notes ("weighted toward Kubernetes + distributed systems because CV gap")

Proceed without waiting for approval unless the user asked to review it. (If the request is
ambiguous in a way that changes the whole app, ask - otherwise decide and go.)

## Step 2 - Scaffold

Run (adapt flags to what the installed `create-next-app` accepts; check `--help` first):

```bash
npx create-next-app@latest <slug> --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --turbopack --yes
cd <slug>
npm install @prisma/client@^6 react-markdown remark-gfm react-syntax-highlighter
npm install -D prisma@^6 @types/react-syntax-highlighter
```

Pin Prisma to **v6** (the templates use `prisma-client-js` and `env("DATABASE_URL")`; Prisma 7
changed the client/config model). If you must use a newer Prisma, port the `schema.prisma`
generator + `prisma.ts` accordingly and keep behaviour identical.

Then:

1. `.env.local`:
   ```
   OPENAI_API_KEY=
   OPENAI_MODEL=gpt-4o-mini
   ```
   `.env` (Prisma CLI reads `.env`, not `.env.local`): `DATABASE_URL="file:./dev.db"`
2. Append to `.gitignore`: `/prisma/dev.db` and `/prisma/dev.db-journal` (history is personal).
3. Write `prisma/schema.prisma` (see `reference/architecture.md`), then `npx prisma migrate dev --name init`.
4. Delete the default boilerplate (`public/*.svg`, default `page.tsx` content).
5. Read `node_modules/next/dist/docs/` as per Step 0.

## Step 3 - Write the application

Follow `reference/architecture.md` + `reference/code-templates.md` + `reference/design-system.md`.
Substitute the app-specific values (title, domains, persona) from your plan. Files to produce:

```
prisma/schema.prisma
src/lib/{prisma,quiz,actions,chatActions,stats}.ts
src/data/{types,examplePrompts}.ts
src/data/questions/{index.ts, <one file per domain>.ts}
src/components/{Nav,Markdown,MessageInput,DiscardAttemptButton}.tsx
src/app/{layout.tsx,page.tsx,globals.css}
src/app/quiz/[attemptId]/{page.tsx,QuizRunner.tsx}
src/app/results/[attemptId]/page.tsx
src/app/history/page.tsx
src/app/chat/{layout.tsx,page.tsx,ChatPanel.tsx,ConversationList.tsx}
src/app/chat/[id]/page.tsx
src/app/api/ai/route.ts
README.md
```

The app must have all these capabilities (the user should not have to ask for any):

- Home: question count input, domain checkboxes with per-domain counts, Start Quiz; resume/discard banner if an attempt is in progress (only one active attempt at a time, enforced server-side).
- Quiz runner: progress bar, running score, domain + "Select one / Select all that apply" pills, three question types, per-option state colours after submit, correct/incorrect feedback with explanation, number-key + Enter keyboard shortcuts, resume mid-quiz.
- Results: big colour-coded score, per-domain breakdown table sorted weakest first.
- History: aggregated weak areas across all attempts (bar + %), list of past attempts.
- AI (when `OPENAI_API_KEY` set): "Explain further" + follow-up thread under each answer, and a standalone `/chat` with saved conversations, example prompts biased to weak domains, truncation detection + Continue button.
- No auth, single-user, local SQLite.

## Step 4 - Write the question bank (the biggest part)

Read `reference/question-bank.md` fully. Key rules:

- One `.ts` file per domain exporting `Question[]`; combine in `index.ts` with the duplicate-id guard.
- Mix: ~75-80% `single`, ~8-12% `multi`, ~8-12% `true-false`. Vary correct-answer position.
- Difficulty mix calibrated to the learner's level (beginner 50/40/10, intermediate 25/50/25, advanced 10/40/50 for easy/medium/hard).
- Scenario-based questions ("A team ... what should they do?") should be at least a third of the bank for professional audiences.
- Every question has a teaching explanation (why right, why the tempting wrong answer is wrong).
- **Personalisation:** when a CV/profile exists, tag each domain with the professions it serves, weight the bank toward the *target* roles and the learner's *gaps*, and include a dedicated domain (or two) for the target-role specifics (e.g. system design for staff-eng targets, interview-style scenario questions). Aim for the Maximum size the user selected.
- Write in batches of ~15-25 questions per file with `Write`; for large banks write each domain file in a separate step so quality stays high. If you have subagent tooling and the bank is Large/Maximum, you may fan out one subagent per domain **with the schema + quality bar from `reference/question-bank.md` pasted into its prompt**, then you review/merge. Otherwise write them yourself.
- Facts must be right. If unsure of a version-specific detail, test the concept more generally or use `WebSearch`/docs to check. Do not invent flags, APIs or numbers.

Also write `examplePrompts.ts`: 2 hand-written "good question to ask a tutor" prompts per domain.

## Step 5 - Verify (do not skip)

1. `npx tsc --noEmit` and `npm run lint` - fix everything.
2. Run a **question-bank validator** (a throwaway script via `npx tsx`, or a node script; see `reference/question-bank.md` "Validation"). It must confirm: unique ids, every `single` has exactly 1 correct, every `multi` has >=2 correct, every `true-false` has exactly the options `true`/`false` and 1 correct, every question has non-empty prompt/explanation, domain keys all valid, per-domain counts match plan.
3. `npm run build` must succeed.
4. Start `npm run dev` (background), and exercise it: use the `run` skill or Chrome tools if available, otherwise `curl` the pages (`/`, `/history`, `/chat`) and expect 200. Start a quiz, answer, finish, see results. If `OPENAI_API_KEY` is empty, confirm the AI UI degrades gracefully (button hidden, chat shows the setup hint).
5. Stop the dev server.

If anything fails, fix and re-run. Report honestly what you did and did not verify.

## Step 6 - Hand-off message

Tell the user (concisely):
- Folder path, how to run: `cd <slug> && npm run dev` -> http://localhost:3000
- Where to put the OpenAI key (`.env.local`), that AI features are off without it, and the key advice from `reference/openai-key-setup.md`: create a **Restricted** key with only **Chat completions (`/v1/chat/completions`)** set to **Request**, everything else None, plus a low billing limit
- Copy the substance of `reference/openai-key-setup.md` into the generated app's `README.md` as an "OpenAI API key" section
- Domains and question counts (a small table), and what was personalised from their CV/description
- How to add questions (edit a domain file; ids unique; validator command)
- Anything not verified (e.g. AI calls without a real key)

## Quality bars (checklist before declaring done)

- [ ] Fresh-directory install works from `npm install` -> `npx prisma migrate dev` -> `npm run dev`
- [ ] Look & feel matches `reference/design-system.md` (violet accent, neutral surfaces, rounded-2xl cards, dark mode via `dark:` classes)
- [ ] Question counts hit the plan; validator passes
- [ ] No personal data from the CV in source or prompts
- [ ] README explains run, structure, AI setup (including restricted-key permissions), adding questions
- [ ] tsc, lint, build all pass
