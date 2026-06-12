# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Turtle Guard Conservation Portal is a mobile-first Progressive Web App (PWA) for Greek sea turtle conservation fieldwork. It enables researchers and volunteers to log nest sightings, tag turtles, run morning beach surveys, track inventory, and view GPS-mapped nest data. The app connects to a hosted REST backend at `https://turtle-backend-pxcx.onrender.com`.

## Commands

```bash
npm run dev       # Start dev server on port 3000
npm run build     # Production build
npm run lint      # TypeScript type-check only (no ESLint configured)
npm run test      # Run all Vitest tests
npm run preview   # Preview production build
```

Run a single test file:
```bash
npx vitest run tests/Database.test.ts
```

### Environment Setup

Copy `.env.example` to `.env.local` and set:
```
GEMINI_API_KEY=your_key_here
```

Both `process.env.API_KEY` and `process.env.GEMINI_API_KEY` are injected at build time via `vite.config.ts` — do not use `import.meta.env` for these.

## Architecture

### Routing & State — `App.tsx`

There is no router library. Navigation is controlled entirely by the `AppView` enum (defined in `types.ts`). `App.tsx` holds all top-level state and passes it down as props:

- `view` — which screen is rendered
- `user` — logged-in `User | null`
- `beaches` — fetched once on mount from the backend
- `surveys` — `Record<string, SurveyData>` keyed by beach name; kept in memory for the session
- `selectedNestId` / `selectedTurtleId` — set before navigating to detail screens
- `headerActions` / `headerTitle` — child screens can inject content into the shared header via `setHeaderActions` / `setHeaderTitle` props

The `navigate(view, origin?, date?)` function sets `view` and closes the sidebar. There is no URL-based routing.

### Screens — `screens/`

Each file is a self-contained page component. Screens receive navigation via `onNavigate: (view: AppView) => void` and access data either through props passed from `App.tsx` or by fetching directly from `DatabaseConnection` inside their own `useEffect` hooks.

### API Layer — `services/Database.ts`

All backend communication goes through the `DatabaseConnection` object (a plain object of async functions, not a class instance). The base URL is `API_URL = 'https://turtle-backend-pxcx.onrender.com'`. There is no auth token — authentication is handled by the login call returning user data that is stored in React state.

`decodeProfilePicture()` handles the backend's inconsistent profile picture formats (raw bytes, base64 strings, byte arrays, JSON-wrapped values).

### AI Feature — `components/NestAIQuery.tsx`

Uses `@google/genai` (Gemini API) directly from the browser with `process.env.GEMINI_API_KEY`. It sends nest records as JSON context and returns either a text answer or a chart specification rendered by Recharts.

### Styling

Tailwind CSS v4 (configured via `@tailwindcss/vite` plugin, not `tailwind.config.js`). Theme tokens are defined in `src/index.css`:
- Custom colors: `bg-background-dark` (`#101922`), `bg-background-light` (`#f6f7f8`), `bg-surface-dark` (`#1a232e`), primary blue `#137fec`
- Custom fonts: Inter (sans), Space Grotesk (display), JetBrains Mono (mono)
- Utility class `.glass-panel` for frosted-glass cards

Dark/light mode is toggled by adding/removing the `dark` class on `<html>`. The HTML starts with `class="dark"`.

### Component Library — `components/ui/`

Eight base components (Button, Card, Input, Modal, Select, Textarea, Typography, MetricInput) act as the design system. Use these instead of raw HTML elements. All accept a `theme` prop (`'light' | 'dark'`) where needed.

### Path Alias

`@/` resolves to the project root (e.g., `@/components/ui/Button`).

## Key Types — `types.ts`

- `AppView` enum — every valid screen state
- `User` — authenticated user; `role` field is a free string from the backend
- `NestRecord` — uses `id` as the display nest code; `dbId` is the numeric primary key used for backend calls
- `SurveyData` — in-memory morning survey state per beach; `nests` and `tracks` arrays are appended during a session
- `TurtleRecord.measurements` — all morphometric fields are optional

## Testing

Tests use Vitest + `@testing-library/react` with jsdom. The setup file is `src/setupTests.ts` (imports `@testing-library/jest-dom`). Test globals (`describe`, `it`, `expect`, `vi`) are available without imports.
