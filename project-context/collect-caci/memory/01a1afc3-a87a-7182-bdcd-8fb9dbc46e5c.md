---
nexus: 1
id: 01a1afc3-a87a-7182-bdcd-8fb9dbc46e5c
kind: note
title: collect_caci — overview draft
project: 0cf5ed59-0357-4647-aa28-d8ed6258a8b5
created: "2026-11-06T07:29:57Z"
updated: "2026-11-06T07:29:57Z"
source: claude
verification: unverified
labels: [draft, overview]
---

# Collect Your Ćaci: overview

## What it is
A satirical social game for a Serbian audience. Players ("studenti") "catch" Ćaci, caricatures of people they see as pro-government loyalists. Each one gets a name, age and opština (municipality), and the crowd votes plus or minus *(verified: CLAUDE.md, backend/src/routes/cacis.js)*. A ćaci the crowd rejects is stamped "NIJE ĆACI" and costs its collector points *(verified: backend/src/services/scoring.js)*. The UI is Serbian (Latin script); code, comments and commits are in English *(verified: CLAUDE.md)*.

## Who uses it
- Players on Android, iOS and the web (one Flutter app) *(verified: app/pubspec.yaml, CLAUDE.md)*.
- Two developers ("Marko", "gale_091"), each working with their own Claude. They hand work over through `docs/CHANGELOG.md` *(verified: docs/CHANGELOG.md; 2 authors in git history)*.
- Admins, through a local-only React admin panel *(verified: admin/vite.config.ts, CLAUDE.md)*.

## Main features
- Adding a ćaci. Only the "caricature" mode is open: the user draws it in ChatGPT with one of 50 prompt variants and uploads it *(verified: caciSchema in routes/cacis.js)*.
- Duplicate detection from name trigrams, a local CLIP image embedding and age *(verified: routes/cacis.js, services/similarity.js)*.
- Voting, threaded comments with likes, reports, friends, search, a feed with filters *(verified: routes/cacis.js route list)*.
- Rankings by collector, ćaci and opština, plus a map of Serbia, Montenegro and Republika Srpska *(verified: CLAUDE.md, app/lib/screens)*.
- The indeksi currency: an append-only ledger, daily caps, levels and achievements *(verified: services/economy.js)*.
- Mini games: "Jahanje Ćacija" (ride, in two modes) and Ćacilend (a maze). Scores are client-reported and capped *(verified: routes/games.js)*.
- A store, currently closed *(per CLAUDE.md; not opened)*.
- Email verification with a 6-digit code. Social login is coded but hidden on the beta *(verified: docs/CHANGELOG.md, session.dart)*.

## How to run and test
- `docker compose up -d --build` starts db, api, web and admin. Web runs on :8080, API on :3000, Postgres on :5433 *(verified: docker-compose.yml)*. Demo login: `demo` *(password in README)*.
- `cd app && flutter run -d chrome --dart-define=API_URL=http://localhost:3000` gives hot reload *(verified: README.md)*.
- Checks: `flutter analyze` must stay clean; smoke-test the API with curl as shown in CLAUDE.md *(verified: CLAUDE.md)*.
- **There are no automated tests and no CI.** `app/test/` is empty, there is no backend test script, and there is no `.github/` *(verified: directory listing, backend/package.json)*.

## Status and gaps
- Public beta on Coolify, seeded with demo data. API at caci.mrh.lol *(verified: admin/vite.config.ts, docs/DEPLOY.md)*.
- Still open: a moderation UI for reports, push notifications, password reset, profile editing, a production domain (needed for Apple login), and a per-IP limit on sign-ups *(verified: CLAUDE.md "Open ideas", CHANGELOG 2026-11-05)*.
- The photo and placeholder modes and the store are switched off but still in the code *(verified: routes/cacis.js; store per CLAUDE.md)*.
