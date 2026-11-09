# briefly

**Personalized news, delivered as a feed you can read or a briefing you can listen to.**

briefly turns a user's interests, location, follows, and feedback history into a curated selection of recent news stories and a **5, 10, or 20 minute** audio news briefing.

Built by Ekaansh Agrawal, Lakshya Wahi, Cheren Song, and Devon Parikh as a submission to the Optimization Track of the 2026 HackCMU 24-hour hackathon.

---

## Features

- **Onboarding profile.** Pick from 68 interests across 8 groups, then add a home city and country, sports teams, a stock watchlist, other places, companies, people, and topics to skip.
- **Fresh, profile-aware feed.** Each profile category (places, sports, markets, following, interests) is turned into Google News RSS queries covering the last 3 days. Results are deduplicated, filtered against excluded topics, and interleaved round-robin so no single interest dominates.
- **A feed that learns.** Rate stories from 1 to 5 stars. A small per-user pairwise logistic regression model (headline keywords, category, publisher, recency, and a multi-publisher coverage signal) re-ranks the feed. After enough new ratings, the model also suggests discovery keywords to search for.
- **Gemini summaries.** Stories are summarized (roughly 55 to 160 words each) with Google Gemini, cached, and generated in batches.
- **Timed briefings.** Choose 5, 10, or 20 minutes. briefly picks the top-ranked stories (8, 18, or 40), has Gemini write a sectioned script targeting about 150 words per minute, and checks the length, retrying once if it is off by more than 10%.
- **Sourced, validated charts.** Briefs can include bar, line, area, pie, or donut charts. A chart is only shown if every data point is backed by an exact evidence excerpt from a source summary, with matching label, number, and unit.
- **Audio narration.** Scripts are chunked and voiced with ElevenLabs, with playback speeds from 0.75x to 2x.
- **Resilient by design.** Feeds are cached for 15 minutes per profile, with a stale-feed fallback (up to 24 hours) when news sources are down. Scripts fall back to headline-based summaries if Gemini is unavailable.

## Tech stack

| Area | Tools |
| --- | --- |
| Framework | Next.js 16 (App Router), React 19, TypeScript |
| Styling / UI | Tailwind CSS 4, shadcn/ui on Base UI, Motion, Lucide icons |
| Charts / maps | visx, d3, Recharts, `world-atlas` (location picker) |
| Database | MongoDB Atlas |
| Auth | `bcryptjs` password hashing, JWT sessions via `jose` in an httpOnly cookie |
| News | Google News RSS (parsed with `fast-xml-parser`) |
| AI | Google Gemini (summaries, briefing scripts, chart extraction) |
| Voice | ElevenLabs text-to-speech |
| Tooling | oxlint, oxfmt, Node's built-in test runner |

## Getting started

### Prerequisites

- Node.js **22.13 or newer**
- A [MongoDB Atlas](https://www.mongodb.com/atlas) cluster (make sure your IP is allowed under Network Access)
- A [Google AI Studio](https://aistudio.google.com/) API key for Gemini
- An [ElevenLabs](https://elevenlabs.io/) API key and voice ID (only needed for audio narration)

### Setup

```bash
git clone https://github.com/csongfm/HackCMU-briefly.git
cd briefly
npm install
cp .env.example .env.local
```

Fill in `.env.local`:

| Variable | Required | Description |
| --- | --- | --- |
| `MONGODB_URI` | Yes | MongoDB connection string (`mongodb://` or `mongodb+srv://`) |
| `MONGODB_DB` | No | Database name (defaults to `briefly`) |
| `AUTH_SECRET` | Yes | Session signing secret, at least 32 characters. Generate one with `openssl rand -base64 32` |
| `GEMINI_API_KEY` | Yes, for summaries and briefs | Google AI Studio key |
| `GEMINI_MODEL` | No | Defaults to `gemini-flash-latest` |
| `GEMINI_USE_GOOGLE_SEARCH` | No | Set to `true` to enable Google Search grounding (needs available quota or billing) |
| `ELEVENLABS_API_KEY` | Yes, for audio | ElevenLabs API key |
| `ELEVENLABS_VOICE_ID` | Yes, for audio | Voice to narrate with |

All keys are server-side only. Never prefix them with `NEXT_PUBLIC_`.

### Run it

```bash
npm run dev
```

Then open [http://localhost:3000](http://localhost:3000), create an account, set up your profile, and open your feed.

### Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the dev server |
| `npm run build` | Production build |
| `npm start` | Serve the production build |
| `npm test` | Run the unit tests in `tests/` |
| `npm run lint` | Lint with oxlint |
| `npm run format` | Format with oxfmt |

## How it works

```
Profile ──► Google News RSS queries ──► curate + dedupe ──► per-user ranking ──► feed
                                                                  ▲                │
                                                       star ratings (learning)     ▼
                                          Gemini summaries ──► briefing script ──► ElevenLabs audio
```

1. **Query building** (`lib/news.ts`). Profile fields are grouped by category and batched into OR queries of up to five terms, each limited to the past three days.
2. **Curation** (`lib/news.ts`). Stories older than 72 hours, duplicates, and anything mentioning an excluded topic are dropped. Categories are interleaved round-robin.
3. **Ranking** (`lib/feed-learning.ts`). Every rating becomes a feature snapshot. Pairs of differently rated stories train a logistic model regularized toward a freshness-first prior, so a brand-new account still sees a sensible, recency-ordered feed.
4. **Briefing** (`lib/briefing.ts`, `lib/gemini-news.ts`, `lib/interactive-brief.ts`). The top stories are summarized, assembled into a sectioned script with source references, and validated before being returned.
5. **Narration** (`lib/narration.ts`, `app/api/voice`). The script is split into sentence-aligned chunks of up to 2,400 characters and sent to ElevenLabs.

## API routes

All routes except auth require a signed-in session. Mutating routes also check the request origin.

| Route | Method | Purpose |
| --- | --- | --- |
| `/api/auth/register` | POST | Create an account |
| `/api/auth/login` | POST | Sign in |
| `/api/auth/logout` | POST | Sign out |
| `/api/auth/me` | GET | Current user and saved profile |
| `/api/profile` | PUT | Validate and save the listener profile |
| `/api/news` | GET | Ranked personalized feed (cached, with fallback) |
| `/api/news/feedback` | POST | Save or clear a 1 to 5 star rating |
| `/api/news/summaries` | POST | Gemini summaries for up to 8 feed stories |
| `/api/news/transcript` | POST | Generate a 5, 10, or 20 minute briefing |
| `/api/news/image` | GET | Look up and cache a story's image |
| `/api/voice` | POST | Text-to-speech via ElevenLabs (up to 5,000 characters) |

## Project structure

```
app/                 Pages (landing/onboarding, feed) and API routes
components/
  briefly/           Brief chart, interactive brief, story image
  bklit/, charts/    Chart and map primitives adapted from Bklit UI
  kokonutui/         Animated background paths
  ui/                shadcn/ui components
lib/                 News fetching, ranking, briefing, Gemini, auth, MongoDB
public/              Fonts, news placeholder images, world map data
tests/               Unit tests (news, ranking, preferences, briefs, and more)
```

## Testing

```bash
npm test
```

Tests cover query building and RSS parsing, curation, the feed-learning model, profile validation, briefing selection, chart validation, and story image handling.

## Credits and licenses

- UI building blocks are adapted from [Kokonut UI](https://kokonutui.com) and [Bklit UI](https://bklit.com), both MIT licensed (see `licenses/`).
- Display font: [Anton](https://github.com/google/fonts/tree/main/ofl/anton) (SIL OFL).
- Map data: [world-atlas](https://github.com/topojson/world-atlas).
- Topic illustration photos are from [Unsplash](https://unsplash.com/license) (see `public/images/news/CREDITS.md`). They illustrate topics and are not photos of the reported events.
- Full source notes are in [`UI_SOURCES.md`](./UI_SOURCES.md).

News headlines are sourced through Google News RSS. briefly links out to publishers for full reporting.
