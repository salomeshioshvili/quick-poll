# Quick Poll

A fast, shareable polling app built with SvelteKit and Firebase Realtime Database. Create a poll in seconds, share the link, and watch results update live.

## Features

- **Create polls** — add a question, any number of options, an optional image, and choose a color theme
- **4 gradient themes** — Violet, Ocean, Sunset, Slate (persists to the poll and results pages)
- **Live results** — votes reflect in real time via Firebase
- **View & share tracking** — session-gated view counter, share counter incremented on copy
- **QR code** — generate a scannable QR for the voting link directly on the results page
- **Image upload** — attach a photo to the poll (base64, max 500 KB)
- **Glassmorphism UI** — consistent frosted-glass design across all three pages

## Pages

| Route | Purpose |
|---|---|
| `/` | Create a new poll |
| `/poll/[id]` | Vote on a poll |
| `/results/[id]` | View live results, share link, or show QR code |

## Tech Stack

- [SvelteKit 5](https://svelte.dev/docs/kit)
- [Firebase Realtime Database](https://firebase.google.com/docs/database)
- Inline SVG icons (no icon library dependency)
- Vite

## Getting Started

### 1. Clone & install

```sh
git clone <repo-url>
cd quick-poll
npm install
```

### 2. Configure Firebase

Create a `.env` file in the project root with your Firebase project credentials:

```env
VITE_FIREBASE_API_KEY=your_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
VITE_FIREBASE_DATABASE_URL=https://your_project-default-rtdb.firebaseio.com
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_FIREBASE_APP_ID=your_app_id
```

> The `.env` file is git-ignored. Never commit credentials.

### 3. Run locally

```sh
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

## Build & Deploy

```sh
npm run build
npm run preview    # preview production build locally
```

Install an [adapter](https://svelte.dev/docs/kit/adapters) for your deployment target (Vercel, Netlify, Node, etc.).
