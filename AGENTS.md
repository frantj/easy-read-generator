# AGENTS.md

Instructions for AI coding agents working on Easy Read Generator. See `README.md` for features, API, deployment, and the Claude skill.

## What the app does

Users paste text (1–30,000 characters). The server makes two OpenAI calls:

1. `buildMessages()` turns the text into Easy Read JSON: `title`, `summary`, `sections[].heading`, `sections[].sentences[].text`.
2. `buildImageSelectionMessages()` picks an image keyword for each sentence from `data/image-map.json`.

The results page shows each sentence with an image on the left and the text on the right. Users can copy, download as HTML, print, or save a shareable link. Shared documents are written to `data/documents/` and deleted 24 hours later.

## Tech rules

- Node.js 18+ and Express on the server. Plain JavaScript in the browser.
- No TypeScript, no front-end frameworks, no build step.
- Client code in `public/js/` uses ES5 style (`var`, an IIFE per file) to match the existing files.
- Call OpenAI with `fetch`. Do not add the OpenAI SDK.
- Helmet sets a strict Content Security Policy: no inline scripts and no third-party scripts. Put new scripts in `public/js/`.
- Ask before adding a dependency.

## OpenAI

- Model: `OPENAI_MODEL`, default `gpt-6-luna`, using Chat Completions with `reasoning_effort: 'none'`.
- GPT-4.x models reject `reasoning_effort`. `callOpenAI()` leaves it out for them, so setting `OPENAI_MODEL=gpt-4o-mini` still works as a rollback.
- If you change the reasoning effort to anything other than `none`, remove `temperature`.
- Responses are parsed with `extractJson()` and checked with `validateEasyReadResponse()`. Malformed JSON triggers one retry.

## Easy Read rules

These are enforced in the prompt in `buildMessages()`:

- One idea per sentence, 15 words or fewer
- Everyday words, active voice, no contractions
- Numbers as figures
- Address the reader as "you" and the organisation as "we"
- Explain any hard word in the next sentence

Image keywords must exist in `data/image-map.json`. Unknown keywords show a text placeholder. Only name catalog keywords in prompt hints.

## Privacy and accessibility

- The API key stays on the server. Never send it to the client or log it.
- Do not log user text or AI output. Only log errors, and only when `NODE_ENV !== 'production'`.
- Keep the rate limits and input validation.
- Use semantic HTML, label every control, and keep focus visible.
- Keep high-contrast colours, large tap targets, and left-aligned text with generous spacing.
- Everything must work with the keyboard. Modals close with Escape.
- Announce loading and status changes with `aria-live="polite"`.

## Keep the Claude skill in sync

`skill/easy-read/SKILL.md` copies the writing rules and image guidance from `server.js` by hand. If you change either prompt, update `SKILL.md` too. See "Keeping the skill in sync with the site" in `README.md`.

## Commands

```bash
npm install
npm run dev                  # http://localhost:3000, needs OPENAI_API_KEY in .env
node --check server.js       # syntax check
python3 skill/package_skill.py   # rebuild the Claude skill zip
```

There is no automated test suite. To verify a change, run the server and send a request:

```bash
curl -X POST http://localhost:3000/api/transform -H 'Content-Type: application/json' -d '{"text": "Your text here."}'
```

Check that the response has a title, summary and sections, and that every `imageKeyword` exists in `data/image-map.json`.
