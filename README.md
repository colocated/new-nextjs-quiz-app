# new-nextjs-quiz-app

A [Claude Code](https://claude.com/claude-code) skill that builds a complete, personalised Next.js
study/quiz app from an empty directory. Claude writes every file and the whole question bank itself.

**What it generates**

- Randomized multi-domain quizzes (single, multi-select, true/false) with explanations
- Score history and weak-area tracking (Prisma + SQLite, single-user, no auth)
- Optional OpenAI tutor: "Explain further" under each answer, plus a saved-conversations chat
- Violet/neutral Tailwind design system with automatic dark mode
- A question bank tailored to your goals, or to your CV / job spec / syllabus if you attach one

## Install

The skill is just a folder. Clone it into Claude Code's skills directory.

**macOS / Linux**
```bash
git clone https://github.com/colocated/new-nextjs-quiz-app.git ~/.claude/skills/new-nextjs-quiz-app
```

**Windows (PowerShell)**
```powershell
git clone https://github.com/colocated/new-nextjs-quiz-app.git "$env:USERPROFILE\.claude\skills\new-nextjs-quiz-app"
```

Restart Claude Code (or start a new session) so it picks up the skill, then check that
`/new-nextjs-quiz-app` appears when you type `/`.

**Project-only install** (shared with a team via the repo):
```bash
git clone https://github.com/colocated/new-nextjs-quiz-app.git .claude/skills/new-nextjs-quiz-app
rm -rf .claude/skills/new-nextjs-quiz-app/.git
```

**Update later:** `git -C ~/.claude/skills/new-nextjs-quiz-app pull`

## Usage

Run it from the directory where you want the new app folder created:

```
/new-nextjs-quiz-app "Java Engineering Course"
/new-nextjs-quiz-app "Java Engineering Course" "I know Python well, new to the JVM; targeting backend interviews"
/new-nextjs-quiz-app "My Career Trainer"     # with a CV, job spec or syllabus attached
```

1. The first argument is the app title (required unless a file is attached). The second is an optional description of what you know or want to target.
2. If you attach a CV or similar, Claude summarises your profile, then asks 3-6 quick questions (goal, target roles, level, question-bank size, AI on/off).
3. It scaffolds the app, writes the question bank, and verifies with type-check, lint, build and a dev-server smoke test.
4. Nothing personal from your CV is written into the generated app.

Then:
```bash
cd <generated-app>
npm run dev        # http://localhost:3000
```
Add `OPENAI_API_KEY=...` to the app's `.env.local` to enable the AI features. Without a key the app
works fully and simply hides them.

## OpenAI API key (optional, for the AI features)

The generated app only calls `POST /v1/chat/completions`, so use a **Restricted** key:

1. platform.openai.com -> API keys -> Create new secret key -> **Restricted**.
2. Leave everything at **None** except Model capabilities -> **Chat completions (`/v1/chat/completions`)** = **Write**.
3. Put it in the app's `.env.local` as `OPENAI_API_KEY=...`. Never commit it.
4. Set a low monthly budget under Billing -> Limits, and use a dedicated Project for the app.

Full guide and best practice: [`reference/openai-key-setup.md`](reference/openai-key-setup.md).

## Requirements

- Claude Code
- Node.js 20+ and npm on the machine that runs the generated app
- Optional: an OpenAI API key

## Repo layout

```
SKILL.md                      procedure, intake questions, quality checklist
reference/architecture.md     stack, data model, routes, behaviour rules
reference/design-system.md    tokens and exact Tailwind class recipes
reference/code-templates.md   full source for every non-content file
reference/question-bank.md    schema, quality bar, personalisation, validator script
reference/ai-integration.md   /api/ai route, ChatPanel, prompts, truncation handling
reference/openai-key-setup.md restricted-key permissions and best practice
```

## License

MIT
