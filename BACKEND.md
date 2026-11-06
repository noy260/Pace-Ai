# PACE AI — what needs a backend

The app in `index.html` is fully static and runs on GitHub Pages. Everything below that is marked **needs a server** cannot be done safely from a static page.

| Feature | Status on GitHub Pages | What it needs |
|---|---|---|
| Plan engine, workouts, paces, adaptation, calendar, stats | Works (runs in the browser) | — |
| Live GPS run + Hebrew/English voice coach | Works while the app is open and the screen is on | A native iOS/Android wrapper for locked-screen / background GPS and audio |
| Data storage | On this device (localStorage) + JSON backup export/restore | A database (e.g. Supabase/Firebase) with auth for cloud sync across devices |
| Strava history | Archive import (zip / `activities.csv`) works | Live sync needs a Strava API app + server-side OAuth token exchange. Note: Strava's API policy forbids using API data in AI applications |
| AI coach | Built-in, clearly labelled answers | A small server that holds the Anthropic API key (below) |
| Push notifications | In-app reminders only | A service worker + Web Push server (VAPID keys), or native push |

## Secure AI coach proxy (not deployed)

The page never contains an API key. To enable the AI coach on GitHub Pages, deploy a tiny proxy that keeps the key server-side, then add this line to `index.html` before the main script:

```html
<script>window.PACE_AI_PROXY = 'https://YOUR-WORKER.workers.dev/coach';</script>
```

The app sends `{ system, messages }` and expects `{ text }` back.

### Example: Cloudflare Worker

```js
// wrangler secret put ANTHROPIC_API_KEY
const ALLOWED_ORIGIN = 'https://noy260.github.io';
export default {
  async fetch(req, env) {
    const cors = { 'Access-Control-Allow-Origin': ALLOWED_ORIGIN, 'Access-Control-Allow-Methods': 'POST', 'Access-Control-Allow-Headers': 'content-type' };
    if (req.method === 'OPTIONS') return new Response(null, { headers: cors });
    if (req.method !== 'POST' || req.headers.get('origin') !== ALLOWED_ORIGIN) return new Response('forbidden', { status: 403 });
    const { system, messages } = await req.json();
    if (!Array.isArray(messages) || messages.length > 20 || JSON.stringify(messages).length > 20000) return new Response('bad request', { status: 400, headers: cors });
    const r = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'content-type': 'application/json', 'x-api-key': env.ANTHROPIC_API_KEY, 'anthropic-version': '2023-06-01' },
      body: JSON.stringify({ model: 'claude-sonnet-5-5', max_tokens: 700, system: String(system).slice(0, 12000), messages }),
    });
    const j = await r.json();
    const text = (j.content || []).map(c => c.text || '').join('');
    return new Response(JSON.stringify({ text }), { status: r.ok ? 200 : 502, headers: { ...cors, 'content-type': 'application/json' } });
  },
};
```

Add rate limiting (e.g. Cloudflare rate-limit rules per IP) before sharing the link publicly, since anyone who can open the page can use the proxy.
