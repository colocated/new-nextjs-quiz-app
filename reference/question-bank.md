# Question bank

The question bank is the product. The app shell is ~1,500 lines; the bank should be several
thousand. Spend most of your effort here, and keep quality high at scale.

## Schema (exact)

```ts
export type QuestionType = "single" | "multi" | "true-false";
export interface QuestionOption { id: string; text: string; correct: boolean }
export interface Question {
  id: string;            // "<domain-key>-<n>", globally unique, e.g. "concurrency-17"
  domain: Domain;        // one of DOMAINS
  type: QuestionType;
  prompt: string;
  options: QuestionOption[];
  explanation: string;
}
```

One file per domain: `src/data/questions/<domain-key>.ts`

```ts
import { Question } from "../types";

export const concurrencyQuestions: Question[] = [
  {
    id: "concurrency-1",
    domain: "concurrency",
    type: "single",
    prompt: "...",
    options: [
      { id: "a", text: "...", correct: false },
      { id: "b", text: "...", correct: true },
      { id: "c", text: "...", correct: false },
      { id: "d", text: "...", correct: false },
    ],
    explanation:
      "Why the right answer is right, and why the most tempting wrong one is wrong.",
  },
];
```

### Type rules (enforced by the validator)

| type | options | correct count | ids |
|---|---|---|---|
| `single` | exactly 4 | exactly 1 | `a`,`b`,`c`,`d` |
| `multi` | 4 (occasionally 5) | 2-3 (never 0, never all) | `a`..`d`/`e`. Prompt ends with `(Select all that apply)` |
| `true-false` | exactly 2 | exactly 1 | `true`, `false` (text `True`/`False`). Prompt is a declarative statement |

Grading is all-or-nothing, so for `multi` every option must be unambiguously right or wrong.
IDs: sequential per domain from 1, no gaps required but no duplicates anywhere.

## Quality bar

A question is only good if all of these hold:

1. **Tests understanding, not trivia.** Prefer "why/what happens/which approach" over "what year/what flag spelling" unless the exact syntax is the skill (CLI/commands/API names are fair game when practitioners really need them).
2. **One defensible answer.** If an expert could argue for two options, rewrite. No "all of the above/none of the above".
3. **Plausible distractors.** Wrong options should be real misconceptions, near-miss commands, or true-but-irrelevant statements - not jokes or absurdities. At least two distractors per question should tempt someone who half-knows the topic.
4. **Option parity.** Similar length and grammatical form across options; the correct answer is not systematically the longest. Distribute correct position uniformly over a/b/c/d.
5. **Teaching explanation** (2-4 sentences): states the right answer's reason, names the misconception behind the best distractor, and where useful gives the exact command/syntax. Use backticks for code. No "Option B is correct because it's B".
6. **Self-contained.** No references to "the above", other questions, or external diagrams.
7. **Accurate and current.** Do not guess version-specific facts. If you are not sure, verify (WebSearch / official docs) or rephrase to the stable concept. Never invent APIs, flags, limits or numbers.
8. **Original.** Written in your own words, to test the same *concepts* as public exam objectives or job skills. Never reproduce a vendor's live exam items or paid course content.
9. **No personal data** from any CV in prompts, options or explanations.

### Scenario questions

For professional audiences, >= 1/3 of the bank should be scenario-based:
"A team ... has [situation/constraint]. What is the best approach / what will happen / what is the root cause?" 
Scenarios should be 1-3 sentences with a concrete constraint (scale, failure, deadline, compliance, cost), and the correct answer should be the *trade-off-aware* one.

### Code in questions

The runner renders prompts and explanations through `<Markdown>`, so you can use inline backticks and
short fenced blocks (```` ```java ````) in prompts. Options are plain text (single line, backticks are
not rendered) - keep code in options to short expressions/commands. For "what does this snippet
output/do?" questions, put the snippet in the prompt as a fence and keep options as outcomes.
Use `\n` escapes inside a template literal or normal string for multi-line prompts.

### Difficulty mix (by self-reported level)

| Level | Easy | Medium | Hard |
|---|---|---|---|
| Beginner | 50% | 40% | 10% |
| Intermediate | 25% | 50% | 25% |
| Advanced | 10% | 40% | 50% |

Difficulty isn't a field in the schema; it's a property of how you write them. Within each domain
file, order roughly easy -> hard so early ids are foundational.

### Type mix (whole bank)

~75-80% `single`, ~8-12% `multi`, ~8-12% `true-false`. Per-domain deviations are fine.

## Choosing domains (the "layout" of the bank)

Domains are the app's top-level taxonomy: they drive the home-page checkboxes, per-domain stats,
weak-area detection and AI example prompts. Choose **6-12**.

How to pick them:

- **Certification / exam named** (e.g. "AWS Solutions Architect Associate", "CKA"): mirror the public exam guide's domains/objectives (use their *topics* as labels; reword, don't copy weightings verbatim as claims). Weight question counts roughly by the published domain weights.
- **Course / subject** (e.g. "Java Engineering Course"): split into learning-path stages and cross-cutting skills - e.g. Language Fundamentals, OOP & Design, Collections & Generics, Concurrency, JVM Internals & Performance, Build/Testing, Frameworks, Data & Persistence, Architecture & Design Patterns, Production & Observability.
- **CV-driven / career-driven**: derive from (a) the target professions' core skill sets, (b) the learner's gaps, (c) a "role readiness" domain per target profession (e.g. "System Design Interviews", "Platform Engineering Scenarios"). Do not create a domain for something the learner is already expert in unless the bank is Maximum size.
- Each domain should support **>= 15 questions** (Standard) / **>= 30** (Large) / **>= 50** (Maximum). Merge domains that can't.
- Keys are kebab-case ASCII; labels are short Title Case (<= 28 chars) so they fit in pills and table cells.

Budget by plan size:

| Bank size | Total | Per domain (9 domains) |
|---|---|---|
| Standard | 120-150 | 13-17 |
| Large | 250-300 | 28-33 |
| Maximum | 400-1000+ | 45-110 |

Write the number you promised. If you can't reach it with quality, say so and deliver fewer
rather than padding with weak questions.

## Personalisation workflow (CV / description driven)

1. **Profile** (from attachments + intake): roles held, stack, strengths, gaps, targets, certs.
2. **Profession map**: list the target professions (e.g. "Senior Backend Engineer", "Platform/SRE", "Solutions Architect"). For each, list 8-15 core competencies a hiring panel would probe. Merge into a single competency list.
3. **Domain assignment**: group competencies into 6-12 domains (see above). Tag each domain in your plan with which professions it serves.
4. **Weighting**: allocate question counts: ~40% to target-role competencies the CV does *not* show (gaps), ~35% to core competencies shared across target roles, ~15% to adjacent/stretch topics, ~10% to strengths (as confidence builders and for interview-depth "hard" questions).
5. **Role-readiness content**: include interview-style and on-the-job scenario questions per target profession ("You're on call and p99 latency doubled after a deploy. What do you check first?"), architecture trade-off questions, and "which tool/approach fits" questions.
6. **Learner context string**: set `APP_CONFIG.learnerContext` to a one-or-two sentence, anonymous summary (skills + direction) so the AI tutor pitches explanations at the right level.
7. **Tell the user** what you inferred and how it shaped the domains/weights in the hand-off message.

If only a short description is given ("I know Python, new to the JVM, targeting backend interviews"),
do the same with a lighter profile and ask the intake questions to fill gaps.

If the user names only a title with no description or attachment, ask the intake questions; default
to a broad, certification-style syllabus at the level they choose.

## Authoring workflow

1. Write the domain list + counts into your plan.
2. For each domain, outline 10-20 subtopics (so questions don't repeat the same fact), then write questions covering each subtopic at mixed difficulty/type.
3. Write the file in chunks (<= ~25 questions per `Write`/`Edit` call keeps output reliable). Start each file with `import { Question } from "../types";` and `export const <camelDomain>Questions: Question[] = [ ... ];`.
4. After each file, run the validator (below). Fix before moving on.
5. After all files: write `index.ts` (import all, spread into `ALL_QUESTIONS`, duplicate-id guard) and `examplePrompts.ts`.
6. Sample 10 random questions and re-read them critically (answer correctness, distractor quality, explanation accuracy). Fix and re-sample until clean.

**Parallelising (optional, Large/Maximum):** spawn one subagent per domain. Give each: the schema,
type rules, quality bar, the domain label + subtopic outline + target count + difficulty mix + the
anonymous learner context, the file path to write, and the instruction to run the validator on its
file. Then you review a sample from each and fix problems. Do not let two agents write the same file.

## Validation script

Create `scripts/validate-questions.ts` (throwaway, may stay in the repo) and run with
`npx tsx scripts/validate-questions.ts` (`npm i -D tsx` if absent):

```ts
import { ALL_QUESTIONS } from "../src/data/questions";
import { DOMAINS } from "../src/data/types";

const errors: string[] = [];
const ids = new Set<string>();
const perDomain: Record<string, number> = {};
const perType: Record<string, number> = {};

for (const q of ALL_QUESTIONS) {
  const where = q.id;
  if (ids.has(q.id)) errors.push(`${where}: duplicate id`);
  ids.add(q.id);
  if (!(DOMAINS as readonly string[]).includes(q.domain)) errors.push(`${where}: bad domain ${q.domain}`);
  if (!q.id.startsWith(q.domain + "-")) errors.push(`${where}: id should start with "${q.domain}-"`);
  if (!q.prompt.trim()) errors.push(`${where}: empty prompt`);
  if (q.explanation.trim().length < 40) errors.push(`${where}: explanation too short`);
  const optIds = q.options.map((o) => o.id);
  if (new Set(optIds).size !== optIds.length) errors.push(`${where}: duplicate option ids`);
  const texts = q.options.map((o) => o.text.trim().toLowerCase());
  if (new Set(texts).size !== texts.length) errors.push(`${where}: duplicate option text`);
  if (q.options.some((o) => !o.text.trim())) errors.push(`${where}: empty option text`);
  const nCorrect = q.options.filter((o) => o.correct).length;
  if (q.type === "single") {
    if (q.options.length !== 4) errors.push(`${where}: single needs 4 options`);
    if (nCorrect !== 1) errors.push(`${where}: single needs exactly 1 correct (has ${nCorrect})`);
  } else if (q.type === "multi") {
    if (q.options.length < 4 || q.options.length > 5) errors.push(`${where}: multi needs 4-5 options`);
    if (nCorrect < 2 || nCorrect >= q.options.length) errors.push(`${where}: multi needs 2..n-1 correct (has ${nCorrect})`);
    if (!/select all that apply/i.test(q.prompt)) errors.push(`${where}: multi prompt should say (Select all that apply)`);
  } else if (q.type === "true-false") {
    if (optIds.join() !== "true,false") errors.push(`${where}: true-false options must be ids true,false`);
    if (nCorrect !== 1) errors.push(`${where}: true-false needs exactly 1 correct`);
  } else {
    errors.push(`${where}: unknown type ${q.type}`);
  }
  perDomain[q.domain] = (perDomain[q.domain] ?? 0) + 1;
  perType[q.type] = (perType[q.type] ?? 0) + 1;
}

// Heuristic: correct-answer position bias on singles (should be roughly uniform).
const pos: Record<string, number> = {};
for (const q of ALL_QUESTIONS.filter((q) => q.type === "single")) {
  const id = q.options.find((o) => o.correct)?.id ?? "?";
  pos[id] = (pos[id] ?? 0) + 1;
}

console.log("Total:", ALL_QUESTIONS.length);
console.log("Per domain:", perDomain);
console.log("Per type:", perType);
console.log("Single correct-position spread:", pos);
for (const d of DOMAINS) if (!perDomain[d]) errors.push(`domain ${d} has no questions`);
if (errors.length) {
  console.error(`\n${errors.length} problem(s):\n- ` + errors.join("\n- "));
  process.exit(1);
}
console.log("OK");
```

Note: the runtime shuffles option order per quiz, so position bias in the file is cosmetic, but a
heavily skewed bank (e.g. 70% `b`) usually signals lazy authoring - rebalance.

## Style reference (from the original app)

The original bank (a Terraform Associate trainer, 140 questions across 9 domains: 120 single,
13 true-false, 7 multi) had this feel - match it:

- Prompt: crisp, one sentence where possible. "What is the primary purpose of Terraform's state file?"
- Single with plausible-but-wrong distractors (e.g. "cache downloaded provider plugin binaries", "store a copy of the HCL source").
- True/false statements with a non-obvious truth: "Terraform state can contain sensitive values in plain text." -> True, and the explanation covers why `sensitive = true` doesn't change that.
- Multi: "Which of the following are valid uses of CLI workspaces? (Select all that apply)" with 3 correct and 1 near-miss.
- Explanations teach beyond the answer: name the exact subcommand, the common misconception, and the safe practice.

Example (single):
```ts
{
  id: "state-4",
  domain: "state",
  type: "single",
  prompt: "Why is using a remote backend generally preferred over local state files for team environments?",
  options: [
    { id: "a", text: "Remote backends are always free of charge, while local state files cost money to keep", correct: false },
    { id: "b", text: "Remote backends enable shared access, locking, and often encryption, avoiding conflicting local copies", correct: true },
    { id: "c", text: "A local state file is structurally incapable of storing resource attributes", correct: false },
    { id: "d", text: "Remote backends remove the need to configure any providers at all", correct: false },
  ],
  explanation:
    "A shared remote backend lets a team access a single source of truth for state, supports locking to prevent concurrent conflicting writes, and commonly offers encryption at rest - none of which a local `terraform.tfstate` file provides on its own.",
},
```

The new app should be **deeper** than that original: bigger, more scenario-based, and tailored.
