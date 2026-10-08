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
   | Model capabilities -> **Chat completions (`/v1/chat/completions`)** | **Write** (the level that allows sending requests; some dashboards label it "Request") |
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

## Best practice

- **Least privilege:** one endpoint, as above. A leaked key can then only spend credits on chat completions, not read files, fine-tunes or other org data.
- **Cap spend:** under **Billing -> Limits**, set a low monthly budget and an email alert. Keep prepaid credit small (e.g. $5-10) for a personal study app. This is the real safety net.
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
