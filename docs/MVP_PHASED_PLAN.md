# Tiny Steps MVP – phased plan

**Goal:** Website MVP. No login. Progress and tasks persisted in local storage.

---

## Keeping API keys secret

- **Server only.** The OpenRouter key is used only in `app/api/chunk/route.ts` (API route). That code runs on the server; the key is never sent to the browser. The frontend only calls your own `/api/chunk` endpoint — it never sees or sends the key.
- **Env vars.** Put the real key in `.env.local` and never commit that file. Phase 1 already has `.env.local` in `.gitignore`. Use `.env.local.example` with a placeholder only (e.g. `OPENROUTER_API_KEY=your_key_here`); no real keys in the repo.
- **No `NEXT_PUBLIC_`.** In Next.js, only vars starting with `NEXT_PUBLIC_` are exposed to the client. Keep your key as `OPENROUTER_API_KEY` (no prefix) so it stays server-side.
- **Deployment.** When you deploy (e.g. Vercel), set `OPENROUTER_API_KEY` in the host’s environment/dashboard. Do not put production keys in code or in the repo.

---

## Phase 1: Project and shell

**Outcome:** Run a Next.js app locally with a minimal layout and one route.

1. **Scaffold Next.js app**
   - Create Next.js project (App Router, TypeScript, Tailwind).
   - Add `.env.local.example` with `OPENROUTER_API_KEY` (for [OpenRouter](https://openrouter.ai) AI calls).
   - Add `.gitignore` for `.env.local` and `node_modules`.
   - Run `npm run dev` and confirm the app loads.

2. **Base layout and styling**
   - Single layout: header with “Tiny Steps” (or logo), calm palette, readable font.
   - Global styles: sentence case for UI copy, no harsh reds for “skip” or “parked”.
   - Optional: simple favicon.

3. **Local storage helper**
   - Add a small `lib/storage.ts` (or similar): `getItem`, `setItem`, `removeItem` wrappers around `localStorage` with a key prefix (e.g. `tinysteps_`) and safe JSON parse/stringify.
   - Use this everywhere you read/write app data so keys and structure stay consistent.

**Checkpoint:** App runs; layout looks like Tiny Steps; storage helper exists and is used nowhere yet.

---

## Phase 2: List input and first chunk (API + UI)

**Outcome:** User can submit a to-do list and see one AI-generated micro-step.

4. **Chunk API route**
   - `app/api/chunk/route.ts`: POST handler.
   - Accept body: `{ list: string, energy?: number }` for “first chunk”, or `{ list: string, currentChunk: string, reason: "smaller" }` for “break down this chunk”.
   - Call **OpenRouter** (`https://openrouter.ai/api/v1/chat/completions`): use `OPENROUTER_API_KEY` from env; request format is OpenAI chat-completions compatible. Optional: set model via `model` in the request body (e.g. `openai/gpt-4o-mini` or another [OpenRouter model](https://openrouter.ai/docs#models)).
   - Use prompts from `lib/prompts.ts`:
     - From list: “Break this list into very small steps. Return only the first step as a single short sentence (e.g. ‘Drink a glass of water. 1 minute.’).”
     - From “smaller”: “The user said this step is too big. Break it into one smaller step. Return only that one step.”
   - Return JSON: `{ chunk: string, reason?: string }`.
   - Handle errors (e.g. 500 + message); no API key in frontend.

5. **Prompts module**
   - `lib/prompts.ts`: export `getFirstChunkPrompt(list, energy?)` and `getSmallerChunkPrompt(currentChunk)`.
   - Keep tone: matter-of-fact, non-judgmental, time-boxed where helpful (“2 minutes”, “one item”).

6. **List screen**
   - Route: e.g. `app/page.tsx` or `app/list/page.tsx`.
   - Form: textarea “What do you need to do?” (placeholder: e.g. “do laundry, reply to mum, eat something”).
   - Optional: “How’s your energy today?” 1–5 (buttons or select).
   - Submit: POST to `/api/chunk` with `{ list }` (and optional `energy`).
   - On success: store in state (and later in local storage) the `list`, `energy`, and the returned `chunk`; navigate or show chunk view (Phase 3).

**Checkpoint:** User can enter a list, get one chunk back from the API, and see it in the UI (even if “chunk view” is still minimal).

---

## Phase 3: Chunk view and core loop (smaller / done / skip)

**Outcome:** User sees one chunk at a time and can choose “Smaller”, “Done”, or “Skip”.

7. **Chunk screen / component**
   - Show the current chunk as the main content.
   - Three actions: **Smaller**, **Done**, **Skip**.
   - Copy: “You can stop after 2 minutes and it still counts” or similar (optional).
   - **Smaller:** POST to `/api/chunk` with `{ list, currentChunk: chunk, reason: "smaller" }`; replace current chunk with response; stay on chunk view.
   - **Done:** Record completion (state + local storage); show a brief reward (Phase 4); then show “next chunk” or “no more chunks”.
   - **Skip:** No penalty; show “next chunk” or “no more chunks” (copy: e.g. “Parked for later”).

8. **“Next chunk” from same list**
   - After Done or Skip, you need the next micro-step. Options:
     - **A)** Call API again with “list + already completed chunks”; prompt: “Given this list and these completed steps, return only the next single step.” Persist `completedChunkTitles[]` in state + local storage.
     - **B)** First call returns an array of chunks (e.g. 5–10); store in local storage; “next” = pop or index+1; “smaller” still calls API for sub-step of current.
   - Choose one approach and implement so “Done” / “Skip” leads to next chunk or “No more steps for now.”

9. **Wire list + chunk to local storage**
   - When user submits list: save `list`, `energy?`, and initial `chunk` (and if using B, the full chunk array).
   - When user completes or skips: save `completedChunkTitles`, `currentChunkIndex` (or equivalent), and `lastUpdated` (timestamp).
   - On app load: if local storage has active list + current chunk, show chunk view instead of list (so refresh doesn’t lose place).

**Checkpoint:** User can go list → first chunk → Smaller (get sub-step) or Done/Skip (get next or “no more”); state survives refresh via local storage.

---

## Phase 4: Progress and rewards (local storage)

**Outcome:** Completions are persisted; user sees simple rewards and “today’s wins”.

10. **Progress model in local storage**
    - Define a simple shape, e.g.:
      - `tinysteps_list` – current list text.
      - `tinysteps_energy` – optional number.
      - `tinysteps_chunks` – current chunk or array of chunks (depending on Phase 3).
      - `tinysteps_completed_today` – array of `{ title: string, completedAt: string }`.
      - `tinysteps_skipped_today` – optional, same shape if you want “parked” list.
      - `tinysteps_last_updated` – ISO string.
    - Use your storage helper; parse safely; reset “today” at midnight (local date) if you want “today’s wins” to be day-scoped.

11. **Reward on Done**
    - When user taps Done: append to `tinysteps_completed_today`; show a short reward (e.g. “Chunk done. +1”, or a single sticker/icon); then show next chunk or “no more” message.

12. **Today’s wins**
    - Screen or section: “Today’s wins” – count + list of completed chunk titles from `tinysteps_completed_today`.
    - Link from “No more steps for now” and/or from header/footer so user can see progress without logging in.

**Checkpoint:** Completions persist; user sees a reward on Done and can view “today’s wins” from stored data.

---

## Phase 5: Polish and MVP cut

**Outcome:** MVP ready for your sister to use; copy and behaviour feel consistent.

13. **Copy and tone pass**
    - All UI in sentence case.
    - List screen: “What do you need to do?”, “Get my first step”.
    - Chunk: “Smaller”, “Done”, “Skip”; “Parked for later” for skip; “No more steps for now” when list is exhausted.
    - No “failed” or guilt; optional short line under chunk about time-boxing.

14. **Empty and error states**
    - List screen: if user submits empty list, show message (“Add at least one thing to do”) and don’t call API.
    - API error: show friendly message (“Something went wrong. Try again.”) and optionally retry button.
    - If local storage is full or disabled: app still works; progress just won’t persist (optional message in footer).

15. **MVP checklist**
    - [ ] Enter list → get first chunk.
    - [ ] Smaller → get sub-step; Done → reward + next or “no more”; Skip → next or “no more”.
    - [ ] Progress and list survive refresh (local storage).
    - [ ] Today’s wins visible.
    - [ ] No login; all data in browser only.
    - [ ] Runs locally; ready to deploy (e.g. Vercel) when you are.

---

## After MVP (out of scope for this plan)

- Voice input (Web Speech API or paste-from-dictation).
- Sticker sets / gamification beyond “today’s wins”.
- Auth, accounts, or sync across devices.
- Mobile app (same API and flow can drive it later).

---

## Summary

| Phase | Focus |
|-------|--------|
| 1 | Project, layout, local storage helper |
| 2 | Chunk API, prompts, list screen, first chunk |
| 3 | Chunk view, Smaller/Done/Skip, next chunk, persist list + chunk |
| 4 | Progress model, reward on Done, today’s wins |
| 5 | Copy, errors, MVP checklist |

No login; progress and tasks live in local storage only.
