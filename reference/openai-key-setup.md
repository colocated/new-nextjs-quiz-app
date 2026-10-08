# OpenAI API key setup and permissions

The generated app makes exactly **one** kind of OpenAI call: a server-side `POST` to
`https://api.openai.com/v1/chat/completions` from `src/app/api/ai/route.ts`. It never lists models,
uploads files, uses embeddings, images, audio, fine-tuning, assistants/threads or the Responses API.
So the key only needs one permission.

## Recommended: a restricted key

1. Go to the OpenAI dashboard -> **API keys** -> **Create new secret key** (https://platform.openai.com/api-keys).
2. Give it a descriptive name, e.g. `quiz-app-local`.
3. Choose the **Restricted** tab (not *All*, not *Read only*).
4. Leave every permission at **None**, except open **Model capabilities** and set:

   | Permission | Setting |
   |---|---|
   | Model capabilities -> **Chat completions (`/v1/chat/completions`)** | **Request** |
   | Everything else (Agents, Traces, Vaults, Voices, List models, Responses, Decisions, Text-to-speech, Realtime, Live, Embeddings, Images, Moderations, Threads, Evals, Fine-tuning, Files, Videos, ...) | **None** |

5. Create the key and copy it immediately (it is shown once).
6. Put it in the app's `.env.local`:
   ```
   OPENAI_API_KEY=sk-...
   OPENAI_MODEL=gpt-4o-mini
   ```
   Restart `npm run dev` after changing env files.

Do **not** pick *Read only*: it cannot make chat completion requests, so the AI features would fail with a 403 (the app shows this as an "OpenAI error" message). Do **not** pick *All* unless you are debugging.

If requests fail with a permission / 403 error mentioning a missing scope, the dashboard's
permission list may have changed since this was written: grant the scope named in the error, and
nothing more.

## Which model it uses, and how to change it

The app reads the model from the `OPENAI_MODEL` environment variable in `.env.local` (see
`src/app/api/ai/route.ts`). If it is unset it falls back to the legacy `OPENAI_EXPLAIN_MODEL`, and
then to the built-in default **`gpt-4o-mini`**: cheap, fast and plenty for short explanations.

To change it later:

1. Edit `.env.local`, e.g. `OPENAI_MODEL=gpt-4.1-mini`.
2. Restart `npm run dev` (Next.js only reads env files at startup).
3. No key permission change is needed; any model reachable through Chat completions works. If you have restricted the Project's allowed models, add the new one there too.
4. If a model rejects the request (some reasoning models do not accept `temperature`, or want a different token parameter), remove `temperature: 0.4` from the request body in `route.ts`. Check the model's page in the OpenAI docs for its supported parameters.

Bigger models give better explanations but cost more and are slower. For a study app, the default is usually the right trade-off.

## Free usage via data sharing (usually covers a quiz app entirely)

OpenAI runs an opt-in program that gives organizations **free daily tokens** on traffic they agree to
share with OpenAI. A personal quiz app uses a tiny amount (a few hundred to a couple of thousand tokens
per "Explain further" or chat message), so with sharing on it will typically never be billed.

To turn it on, an **organization owner** opens
https://platform.openai.com/settings/organization/data-controls/sharing and enables sharing. If your
org is eligible you will see "You're enrolled for complimentary daily tokens".

What to know (as of when this was written; OpenAI can change the terms, so trust the page above):

- **Allowance:** up to 1M tokens/day on the larger models and up to 10M/day on the smaller models. On usage tiers 1-2 (typical for new accounts) that is **250k and 2.5M** respectively. `gpt-4o-mini`, the app's default, is in the smaller-model group.
- **Per project:** you can share only selected Projects, and only usage in those Projects qualifies. Create the key inside a shared Project.
- **You still need a positive balance** on the account. Free tokens show on the Usage page but not as a cost.
- **Not eligible:** Enterprise orgs and Zero Data Retention orgs; fine-tuned models, evals and tool use are excluded. If you do not see the eligibility banner, you are not eligible.
- **Privacy trade-off:** sharing means OpenAI may use your prompts and responses (including your chat questions and the question text sent as context) to improve its models. Do not enable it for a Project if you will paste anything sensitive or private into the app. The generated app is designed to send no personal CV details (see `learnerContext`), but anything you type into the chat is sent. You can switch sharing off at any time on the same page.

## Best practice

- **Least privilege:** one endpoint, as above. A leaked key can then only spend credits on chat completions, not read files, fine-tunes or other org data.
- **Cap spend** (even with free tokens, in case you exceed the allowance or switch model): under **Billing -> Limits**, set a low monthly budget and an email alert. Keep prepaid credit small (e.g. $5-10) for a personal study app. This is the real safety net.
- **Use a project:** create a dedicated OpenAI *Project* for the app and create the key inside it, so usage and limits are isolated from your other work. Optionally restrict the project's allowed models to the one you use.
- **One key per app/machine,** never reuse a key across projects. Name keys so you can revoke the right one.
- **Never commit it.** The generated `.gitignore` excludes `.env*`; keep the key only in `.env.local`. Do not put it in a `NEXT_PUBLIC_*` variable (that would ship it to the browser) and do not paste it into chats, issues or screenshots. If it leaks, revoke it in the dashboard at once and create a new one.
- **Local only:** the app has no auth, so `/api/ai` is open to anyone who can reach the server. Run it on `localhost`. If you ever deploy it publicly, add authentication and rate limiting first, or anyone could spend your credits.
- **Rotate** the key occasionally and delete keys you no longer use. The dashboard shows each key's last-used date.
- **Model choice:** `gpt-4o-mini` is cheap and fine for explanations. Change `OPENAI_MODEL` to trade cost for quality; no permission change is needed.

## Checking that it works

```bash
curl -s -X POST http://localhost:3000/api/ai \
  -H "content-type: application/json" \
  -d '{"messages":[{"role":"user","content":"Say hi"}]}'
```
Expect `{"content":"...","truncated":false}`. A `501` means the key is not set (restart the dev
server); a `502` with an OpenAI error means the key, permission, billing or model name needs fixing.
