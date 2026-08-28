# Study: Chat HTML Interface

## Concept

A chat UI is really just three things: a scrollable list you append DOM nodes to, an input+submit that reads a value and clears itself, and a `fetch()` call that sends the value somewhere and appends whatever comes back. Frameworks (React etc.) exist to manage this at scale, but for one screen, plain JS is clearer to learn from — you'll see every DOM operation instead of it being hidden behind a framework's re-render cycle.

Key browser concepts this phase touches, if any are unfamiliar:
- `fetch(url, {method, headers, body})` returning a `Promise`, `async/await` to consume it.
- Basic DOM manipulation: `document.createElement`, `appendChild`, `scrollTop = scrollHeight` (to auto-scroll to the newest message).
- Disabling the input while a request is in flight (a real UX detail interviewers notice).

## Study material

- **Video:** [Build a Chat App with HTML, CSS and Vanilla JavaScript](https://www.youtube.com/watch?v=_sxoqRIbW0c) — closest match to what Phase 2 needs (no framework).
- **Video (alternative/second opinion):** [Chat Application Using HTML CSS and JavaScript](https://www.youtube.com/watch?v=E6I4wZNSNpU)
- **Written walkthrough with full code:** [Create Working Chatbot in HTML CSS & JavaScript — GeeksforGeeks](https://www.geeksforgeeks.org/javascript/create-working-chatbot-in-html-css-javascript/) — good to have open side-by-side while building, since it's skimmable rather than watched start to end.

## What to deliberately skip from these tutorials

Several of the above build in an actual chatbot LLM call from the frontend (calling an API key directly in JS). Don't do that here — in this project, the frontend only ever talks to your own `/ask` FastAPI endpoint (Phase 3+); the OpenAI key stays server-side only, in `.env`, never in `static/`.

## Notes (fill in as you go)

_What ended up in your `index.html`/CSS/JS, any DOM quirks you hit, screenshots if useful._
