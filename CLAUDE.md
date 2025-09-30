# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Twitter clone built with Next.js, TypeScript, Tailwind CSS, and Firebase. Features include authentication, tweets (with images/GIFs), likes, retweets, replies, bookmarks, user profiles, following/followers, and real-time updates.

## Commands

### Development
```bash
npm run dev              # Start dev server on port 80
npm run dev:emulators    # Start dev server + Firebase emulators (requires Java JDK 11+)
npm run emulators        # Start Firebase emulators only
```

### Build & Test
```bash
npm run build            # Build production bundle
npm start                # Start production server
npm test                 # Run Jest tests in watch mode
npm test:ci              # Run Jest tests in CI mode
npm run lint             # Run ESLint
npm run format           # Check formatting with Prettier
```

### Firebase Setup
```bash
npm install -g firebase-tools          # Install Firebase CLI globally
firebase login                         # Login to Firebase (opens browser)
firebase projects:list                 # List available projects
firebase use <project-id>              # Select project to use
```

### Firebase Deployment
```bash
firebase deploy --except functions    # Deploy Firestore rules, indexes, and Storage rules
firebase emulators:start              # Start all Firebase emulators
cd functions && npm run deploy        # Deploy Cloud Functions
```

## Architecture

### Firebase Integration

The app supports **dual backend modes** controlled by `NEXT_PUBLIC_USE_EMULATOR` in `.env.development`:
- When `true`: Connects to local Firebase emulators (auth:9099, firestore:8080, storage:9199, functions:5001)
- When `false`: Connects to Firebase cloud services

Firebase initialization is centralized in `src/lib/firebase/app.ts:48` which exports `db`, `auth`, and `storage` instances.

### Data Model

**Collections structure:**
- `users/` - User profiles with followers/following arrays
- `tweets/` - All tweets with parent references for replies
- `users/{id}/bookmarks/` - User-specific bookmarks subcollection
- `users/{id}/stats/` - User statistics (likes, tweets arrays)

**Type converters** in `src/lib/types/` use Firestore's `FirestoreDataConverter` pattern for type-safe serialization. See `src/lib/firebase/collections.ts` for collection references with converters.

### State Management

**Context providers** (in `src/lib/context/`):
- `AuthContext` - Global auth state, user data, bookmarks. Auto-creates user on first Google sign-in with random username
- `ThemeContext` - User theme preferences (accent color, background)
- `WindowContext` - Window dimensions for responsive behavior

**Data fetching hooks** (in `src/lib/hooks/`):
- `useCollection` - Real-time Firestore collection listener with optional user population
- `useDocument` - Real-time Firestore document listener
- `useArrayDocument` - Handles array-based documents (stats)
- `useCacheQuery` - Memoizes Firestore queries to prevent unnecessary re-subscriptions
- `useInfiniteScroll` - Pagination for tweet feeds

### Path Aliases

TypeScript path mapping (tsconfig.json:17-21):
- `@components/*` → `src/components/*`
- `@lib/*` → `src/lib/*`
- `@styles/*` → `src/styles/*`

### Component Structure

- `src/components/layout/` - Page layouts with auth guards
- `src/components/tweet/` - Tweet display and interaction components
- `src/components/user/` - User profile components
- `src/components/input/` - Tweet composition with image upload
- `src/components/modal/` - Action modals (edit profile, delete tweet, etc.)
- `src/components/sidebar/` - Navigation sidebar
- `src/components/aside/` - Trends and suggestions sidebar

### API Routes

- `/api/trends/available` - Fetch available trending locations from Twitter API
- `/api/trends/place/[id]` - Fetch trends for specific location (requires `TWITTER_BEARER_TOKEN`)

### Testing

Jest configured with Next.js integration. Path aliases mirrored in `jest.config.js:17-21`. Tests run in jsdom environment.

## Key Implementation Details

- **Image uploads**: Stored in Firebase Cloud Storage, URLs saved in tweet `images` array
- **Real-time updates**: All data fetching uses Firestore `onSnapshot` listeners
- **Username uniqueness**: Enforced via `checkUsernameAvailability` utility during signup and profile updates
- **Pinned tweets**: User document has `pinnedTweet` field referencing tweet ID
- **Next.js image optimization**: Disabled (`next.config.js:6`) for Firebase Storage compatibility
- **Cloud Functions**: Optional. Used to sync user stats when tweets are deleted (requires separate deployment)