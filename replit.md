# Overview

This project is a pnpm workspace monorepo using TypeScript, designed to build a social application with a focus on couple-centric features. The application aims to provide a rich interactive experience through features like personalized themes, a year-in-review summary ("Wrapped"), and engaging mini-games. The core purpose is to deepen connections between partners through shared digital experiences, leveraging a robust backend and a dynamic frontend.

Key capabilities include:
- Real-time push notifications.
- Dynamic UI elements like a "Dynamic Island" for contextual alerts.
- Customizable application themes that can be gifted between partners.
- Interactive games such as "Two Truths One Lie" and a "Wrapped" year-in-review.
- User privacy features like "Safe View".

# User Preferences

I prefer clear and concise communication. For coding tasks, I appreciate an iterative development approach with clear explanations of changes. Please ask for confirmation before implementing major architectural changes or significant feature modifications. Do not make changes to files outside the `src` directory unless specifically instructed.

# System Architecture

The project is structured as a pnpm monorepo, utilizing TypeScript for type safety across all packages.

**Core Technologies:**
- **Monorepo Tool:** pnpm workspaces
- **Node.js:** v24
- **TypeScript:** v5.9
- **API Framework:** Express 5
- **Database:** PostgreSQL with Drizzle ORM
- **Validation:** Zod (`zod/v4`) and `drizzle-zod`
- **API Codegen:** Orval (from OpenAPI spec)
- **Build Tool:** esbuild (CJS bundle)

**UI/UX and Design Patterns:**
- **Theming System:** A dynamic theming system (`lib/themes.ts`, `components/ThemeProvider.tsx`, `components/ThemeDecorations.tsx`) allows partners to gift themes to each other. Themes are applied via CSS custom properties on the `:root` element and a `body::before` pseudo-element with `mix-blend-mode: color` to recolor the UI without modifying individual component styles. Six **seasonal themes** (Beach, Christmas, Autumn, Camping, St Patrick's Day, Easter) layer additional `body.theme-{key}` CSS rules in `styles/themes.css` and mount procedural DOM animation containers (snowflakes, leaves, shamrocks, eggs, an animated campfire scene) via `applyThemeAnimations()` inside `applyTheme()`. Two non-seasonal themes still use the React-based `<ThemeDecorations />` component (Birthday balloons/confetti).
- **Dynamic Island:** A persistent UI component (`components/DynamicIsland.tsx`) provides contextual notifications and animations, using CSS classes for spring animations and `requestAnimationFrame` for re-triggering.
- **Signature Entry Animations:** Each major tab features unique CSS keyframe animations, scoped via `data-page` attributes, providing a branded experience upon navigation. These animations are disabled by `prefers-reduced-motion: reduce`.
- **Safe View:** A client-side feature (`src/lib/safeView.tsx`, `src/components/SafeBlur.tsx`) that blurs sensitive content based on a user preference, without syncing to the backend.
- **Age Verification Gate:** A mandatory 17+ gate (`components/AgeGate.tsx`) appears once per device immediately after the Splash Screen and before Login. Primary persistence is `localStorage` (`tether-age-verified = "true"`) for instant skip on return visits. When the user is authenticated the confirmation is also written to `profiles.age_verified` (boolean column, migration `005_age_verified.sql`) so confirmation persists if localStorage is ever cleared. If the Supabase column reads `true` on profile load, the gate is proactively skipped and localStorage is updated. The "Exit" button redirects to `https://www.google.com`. The component uses the app's dark Liquid Glass aesthetic (inline styles, Framer Motion, Playfair Display / Quicksand fonts).
- **Welcome Guide & App Guide:** Initial user onboarding is managed by a first-time welcome guide, which is replaced by a bottom-sheet "App Guide" menu for recurring access to feature explanations. The guide includes a dedicated "Cheat Sheet" entry covering the new profile quick-reference fields.
- **Name-Neutral, Identity-Neutral Voice:** As of April 2026 the app and AI no longer hard-code the original couple's first names. UI strings, trivia/daily questions, and the AI system prompt all address users as "you" / "your partner" / "the two of you". A universal `NAME_NEUTRAL_RULE` prefix is injected into every prompt sent through `lib/tetherAi.ts`. The After Dark deck has also been re-written to be **gender- and orientation-neutral**: every prompt addresses "you" / "me" / "us" / "your partner" with no gendered nouns or orientation-specific terminology, so the deck applies to any couple regardless of identity.
- **Vibe Check Labels:** The "How are you feeling" emotion list is normalised to a single-word label per vibe (no "Feeling …" prefix, no profanity prefix). The "Horny" vibe replaces the older profane label.
- **Edit Profile Sheet Header:** The Edit Profile bottom-sheet has a sticky header that respects `env(safe-area-inset-top)`, so the title and close button remain visible on devices with a notch / Dynamic Island and stay tappable no matter how far the form is scrolled.
- **App Branding:** The Settings "About" card and the Privacy Statement footer are white-label — they no longer carry the original creator's name. They surface only the corporate "DelKen Tech / DelKen" lineage.
- **Profile Cheat Sheet:** Each partner's profile carries a quick-reference Cheat Sheet (`profile_details` columns: `coffee_order`, `shoe_size`, `size_shirt`, `size_shorts`, `ring_size`, `favourite_colour`, `favourite_food`, `favourite_movies_shows`, `music_preference`, `wind_down`). Coffee Order has a one-tap copy action; the rest render as plain field rows. The Spicy Stats edit form and view card no longer surface `position` or `threesome_prefs` (the columns are kept in the DB so historical answers are preserved).
- **Seen-Question Tracker:** A shared `seen_questions` table (keyed by `couple_id` + `game_key`) tracks which questions the couple has already been served across Trivia, Daily Connection (wholesome + naughty pools tracked separately under `daily_connection` / `daily_naughty`), and Couples Corner. The client-side helper (`lib/seenQuestions.ts`) keeps a local cache, hydrates from Supabase on page mount, and `markSeen` upserts in the background. When the unseen pool is exhausted the tracker transparently recycles. Trivia and Daily Connection pick deterministically from the *unseen* pool so both partners converge on the same daily picks.

**Technical Implementations:**
- **Push Notifications:** Implemented via Web Push, with subscriptions stored in a PostgreSQL `pushSubscriptions` table. VAPID keys are managed through environment variables.
- **Tether Wrapped:** A year-in-review feature that dynamically queries existing `love_messages`, `posts`, `date_plans`, and `time_capsules` tables to generate statistics and AI-driven narratives. AI narratives are cached locally.
- **Two Truths One Lie:** A mini-game with a dedicated database schema for rounds and RLS policies for secure gameplay. Game state is derived from database rounds, with local storage for vanity scores.
- **Realtime Updates:** Utilizes Supabase Realtime for syncing theme changes, pulse notifications (via `love_messages`), and game state for "Two Truths One Lie".
- **Authentication/Authorization:** RLS (Row Level Security) policies are heavily used, particularly for theme gifting (allowing a user to update their partner's `app_theme` column) and game mechanics.

# External Dependencies

- **Supabase:** Used for database (PostgreSQL), Realtime subscriptions, and authentication (implicitly via `auth.uid()` in RLS policies).
- **VAPID (Web Push):** For push notifications, requiring `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, and `VAPID_SUBJECT` environment variables.
- **Anthropic API:** Utilized for AI narrative generation within the "Tether Wrapped" feature, proxied through the application's API server (`POST /api/ai/chat`).