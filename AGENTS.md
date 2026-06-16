# AGENTS.md — CurbSide

## 1. Project Context

CurbSide is a cross-platform mobile application built with React Native (Expo) and TypeScript. It is a two-sided marketplace connecting users requesting curbside services with local providers ("Curb Agents") who fulfill those requests in their geographic area.

The app is conceptually adjacent to the existing ScoutDrive marketplace pattern: requesters create a job with location, photos, and a description; nearby providers see the request, accept it, and complete the task with photo confirmation and in-app payment. Initial beta market is Onslow County, NC.

**Core user flows:**
- Authentication (email + magic link via Supabase Auth)
- Requester: create job → upload photos → set location → submit → track status → pay on completion
- Curb Agent: browse nearby jobs → accept → execute → upload completion photos → receive payment
- Real-time status updates via Supabase Realtime
- In-app payments via Stripe Connect (deferred to Phase 2)

## 2. Tech Stack

**Framework & Language**
- React Native via Expo SDK (latest stable)
- TypeScript (strict mode enabled)
- Expo Router for file-based navigation

**Backend & Data**
- Supabase (PostgreSQL + Auth + Storage + Realtime)
- Row-Level Security (RLS) policies enforced on all tables
- Supabase JS client (`@supabase/supabase-js`)

**State Management**
- Zustand for global client state (lightweight, no boilerplate)
- React Query (`@tanstack/react-query`) for server state, caching, and mutations
- DO NOT use Redux, MobX, or Context API for global state

**Styling**
- NativeWind (Tailwind CSS for React Native)
- DO NOT use StyleSheet.create except for dynamic/computed styles that NativeWind cannot express
- DO NOT use Styled Components or Emotion

**Forms & Validation**
- React Hook Form
- Zod for schema validation (shared between client and server-side validation)

**Maps & Location**
- `expo-location` for geolocation
- `react-native-maps` for map display

**Media**
- `expo-image-picker` for photo capture
- `expo-image` for optimized image rendering

**Payments (Phase 2)**
- Stripe Connect via `@stripe/stripe-react-native`

**Testing**
- Jest + React Native Testing Library for unit/component tests
- Detox or Maestro for E2E (TBD — defer until core flows exist)

## 3. Environment Setup

### Prerequisites (assumed present in the Jules VM)
- Node.js 20.x LTS
- npm 10.x
- Git

### Setup Script

Jules MUST run the following commands in order at the start of every task. This is the canonical setup sequence:

```bash
# Install dependencies
npm ci

# Bootstrap env file from example (placeholders only — never commit .env.local)
cp .env.example .env.local

# Verify TypeScript compiles
npx tsc --noEmit

# Verify lint passes
npm run lint

# Verify tests pass
npm test -- --watchAll=false

# Verify web target builds (smoke test for iOS/Android-equivalent compilation)
npm run build:web
```

If `npm ci` fails because `package-lock.json` does not yet exist, run `npm install` instead and commit the resulting lockfile.

### Environment Variables

The following environment variables are required at runtime. For Jules' VM, create a `.env.local` file with placeholder values during setup so that `expo start` does not crash. DO NOT commit real secrets.

```
EXPO_PUBLIC_SUPABASE_URL=https://placeholder.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=placeholder_anon_key
EXPO_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_placeholder
```

Real values are managed outside the repo and injected at build time. Jules will NEVER commit real keys.

The Supabase client throws at construction time if `EXPO_PUBLIC_SUPABASE_URL` or `EXPO_PUBLIC_SUPABASE_ANON_KEY` is missing. Therefore `.env.local` MUST exist before any build, test, or lint command runs. The setup script above handles this via `cp .env.example .env.local`. Do not skip that step.

## 4. Coding Standards

### General
- TypeScript strict mode is non-negotiable. No `any` types except when interfacing with untyped third-party libraries, and even then prefer `unknown` + type guards.
- All async functions must have proper error handling. No silent failures.
- All Supabase queries must handle the `{ data, error }` pattern explicitly — never assume success.

### File & Folder Structure
```
/app                    # Expo Router screens (file-based routing)
/components             # Reusable UI components
  /ui                   # Primitive UI (Button, Input, Card)
  /features             # Feature-specific composed components
/lib
  /supabase             # Supabase client + typed query helpers
  /stripe               # Stripe helpers (Phase 2)
  /utils                # Pure utility functions
/hooks                  # Custom React hooks
/stores                 # Zustand stores
/schemas                # Zod schemas (shared validation)
/types                  # Shared TypeScript types
/__tests__              # Jest tests, mirroring source structure
```

### Naming
- Components: `PascalCase.tsx`
- Hooks: `useCamelCase.ts`
- Utilities: `camelCase.ts`
- Zod schemas: `entitySchema` (e.g., `jobSchema`, `userSchema`)
- Types/interfaces: `PascalCase` (prefer `type` aliases over `interface` except for extensible public contracts)

### Component Conventions
- Functional components only. No class components.
- One component per file.
- Props typed inline or via a co-located `type Props = { ... }` declaration.
- Export the component as the default export; export Props as a named export if other modules need it.

### State Management Rules
- Local UI state → `useState` / `useReducer`
- Cross-screen shared client state → Zustand store
- Server data → React Query (`useQuery`, `useMutation`)
- NEVER fetch server data into a Zustand store. React Query is the source of truth for server state.

### Styling Rules
- Use NativeWind utility classes inline: `<View className="flex-1 items-center justify-center bg-white" />`
- Custom colors and spacing defined in `tailwind.config.js` — use semantic tokens (e.g., `bg-primary`, `text-muted`) over raw values when possible.

### Testing Standards
- Every new utility function in `/lib/utils` must have a corresponding Jest test.
- Every component with non-trivial logic must have a React Native Testing Library test covering its primary user interactions.
- Mock Supabase via a typed mock client in `__tests__/__mocks__/supabase.ts`.
- Aim for behavior-driven tests, not implementation tests. Test what the user sees and does, not internal state.

### Commits
- Conventional Commits format: `feat:`, `fix:`, `chore:`, `refactor:`, `test:`, `docs:`
- Subject line ≤ 72 characters
- Body explains WHY, not what

## 5. Command Map

Jules MUST use these exact commands. Do not invent variants.

| Command | What It Does |
|---|---|
| `npm ci` | Clean install of dependencies from `package-lock.json`. Use this in CI / VM setup. |
| `npm install` | Install dependencies and update lockfile. Use only when adding/removing packages. |
| `npm run start` | Launches Expo dev server. Not used by Jules directly. |
| `npm run lint` | Runs ESLint against the entire codebase. Must pass before any PR. |
| `npm run lint:fix` | Runs ESLint with auto-fix. Use proactively before committing. |
| `npm run format` | Runs Prettier across the codebase. |
| `npm test` | Runs the full Jest test suite. Must pass before any PR. |
| `npm test -- --watchAll=false` | Single-run mode for CI / Jules VM. |
| `npm run typecheck` | Runs `tsc --noEmit`. Must pass before any PR. |
| `npm run build:web` | Runs `expo export --platform web` to verify the web target compiles. Acts as a smoke test in the VM since iOS/Android binaries cannot be built in Jules' VM. |

### Pre-PR Verification (MANDATORY)

Before opening any pull request, Jules MUST run and confirm all of these pass:

```bash
npm run typecheck && npm run lint && npm test -- --watchAll=false && npm run build:web
```

If any step fails, fix the issue before opening the PR. Do not open a PR with failing checks and a note that says "please review" — that is unacceptable. See Section 9 "Definition of Done" for the complete pre-PR checklist.

## 6. What Jules Should NOT Do

- Do NOT change the chosen tech stack (no swapping NativeWind for StyleSheet, no introducing Redux, no replacing Supabase).
- Do NOT add new top-level dependencies without justification in the PR description.
- Do NOT generate placeholder/lorem-ipsum content in production code. Use realistic CurbSide-domain examples.
- Do NOT commit `.env`, `.env.local`, or any file containing credentials.
- Do NOT modify `AGENTS.md` itself without an explicit user instruction to do so.
- Do NOT bypass RLS policies by using the service role key on the client. Client only ever uses the anon key.
- Do NOT use `localStorage`, `sessionStorage`, `document`, `window`, or any other browser-only API in application code. React Native does not have these. For persistence use `expo-secure-store` (for secrets) or `@react-native-async-storage/async-storage` (for non-sensitive state). The web build target tolerates these APIs but the iOS/Android targets do not — code that compiles for web but breaks on device is a defect.

## 7. Domain Glossary

- **Requester**: User who creates a curbside job (e.g., "pick up a couch from my curb").
- **Curb Agent**: User who accepts and fulfills jobs.
- **Job**: A single unit of work with location, description, photos, and status.
- **Status states**: `draft` → `posted` → `accepted` → `in_progress` → `completed` → `paid` (or `cancelled` at any point).
- **AOI**: Area of Interest — the geographic radius an Agent operates in.

## 8. Contact / Ownership

- Repo owner: @bocanegrajm
- Primary reviewer: @bocanegrajm
- All PRs require human review before merge. Jules does not auto-merge.

## 9. Definition of Done

A task is complete when ALL of the following are true:

- The pre-PR verification command passes cleanly:
  `npm run typecheck && npm run lint && npm test -- --watchAll=false && npm run build:web`
- New utility functions in `/lib/utils` have corresponding Jest tests.
- New components with non-trivial logic have React Native Testing Library tests covering their primary user interactions.
- The PR description includes:
  - The last 20 lines of verification command output
  - A "How to test locally" section if the change is user-facing
  - A list of any new dependencies with a one-line justification each
  - A "Known limitations" section if any TODOs or deferred work remain
- No new top-level dependencies were added unless justified in the PR description.
- `AGENTS.md` is unchanged unless the task is explicitly to modify it.
- No `.env`, `.env.local`, credentials, build artifacts, or `node_modules` are committed.
- Conventional Commits format is used for all commit messages.
- The PR title follows Conventional Commits format and is ≤ 72 characters.

If any item above is not satisfied, the task is NOT done. Do not open the PR. Fix the issue first.
