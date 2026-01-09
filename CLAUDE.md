# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Development

- `npm run dev` - Start development server on port 3000
- `npm run test` - Run Vitest unit tests
- `npm run test -- path/to/file.test.ts` - Run a single test file
- `npm run test -- --watch` - Run tests in watch mode
- `npm run validate` - Run all tests, linting, and typecheck (use before commits)

### Code Quality

- `npm run lint` - ESLint checking
- `npm run lint:fix` - Fix ESLint issues automatically
- `npm run typecheck` - TypeScript type checking
- `npm run format` - Prettier code formatting
- `npm run precommit` - Complete pre-commit validation

### Database

- `npm run setup` - Generate Prisma client and deploy migrations
- `npm run db:generate-type` - Generate Prisma client after schema changes
- `npm run db:prepare-migration` - Create new database migration
- `npm run db:deploy` - Deploy pending migrations to database

### Build & Production

- `npm run build` - Build for **production**
- `npm run start` - Start production server

### E2E Testing (Playwright)

- `npm run test:e2e:install` - Install Playwright browsers
- `npm run test:e2e:dev` - Run E2E tests with UI mode
- `npm run test:e2e:run` - Run E2E tests headless

## Architecture Overview

**Shelf.nu** is an asset management platform built with React Router v7 (migrated from Remix), React, TypeScript, and PostgreSQL.

**Requirements:** Node.js v22.20.0 or higher (see `engines` in package.json).

### Core Technologies

- **React Router v7** - Full-stack React framework with file-based routing (migrated from Remix)
- **React 19** - Latest React version with concurrent features
- **Prisma** - Database ORM with PostgreSQL
- **Supabase** - Authentication, storage, and database hosting
- **Tailwind CSS + Radix UI** - Styling and UI components
- **Jotai** - Atomic state management
- **Hono** - Server middleware via react-router-hono-server

### Key Directory Structure

```
app/
├── routes/          # Remix file-based routes (using remix-flat-routes)
├── modules/         # Business logic services
├── components/      # Reusable React components
├── database/        # Prisma schema and migrations
├── atoms/           # Jotai state atoms
├── utils/           # Utility functions
└── integrations/    # Third-party service integrations
```

### Route Organization

- `_layout+/` - Main authenticated application routes
- `_auth+/` - Authentication and login routes
- `_welcome+/` - User onboarding flow
- `api+/` - API endpoints
- `qr+/` - QR code handling for assets

## Development Patterns

### State Management

- **Server State**: Remix loaders/actions for data fetching and mutations
- **Client State**: Jotai atoms for complex UI state
- **URL State**: Search params for filters, pagination, and bookmarks

### Data Layer

- **Prisma Schema**: Located in `app/database/schema.prisma`
- **Row Level Security (RLS)**: Implemented via Supabase policies
- **Full-text Search**: PostgreSQL search across assets and bookings

### Component Architecture

- **Modular Services**: Business logic separated into `app/modules/`
- **Reusable Components**: Organized by feature/domain in `app/components/`
- **Form Handling**: Remix Form with client-side validation
- **UI Primitives**: Radix UI components with Tailwind styling

### Key Business Features

- **Asset Management**: CRUD operations, QR code generation, image processing
- **Booking System**: Calendar integration, conflict detection, PDF generation
- **Multi-tenancy**: Organization-based data isolation
- **Authentication**: Supabase Auth with SSO support

### Error Handling Pattern

Follow the consistent error handling pattern used throughout the codebase:

**In Routes (loaders/actions):**
```typescript
export async function loader({ request }: LoaderFunctionArgs) {
  try {
    // Business logic
    return payload({ data });
  } catch (cause) {
    const reason = makeShelfError(cause);
    throw data(error(reason), { status: reason.status });
  }
}
```

**In Services:**
- Always throw `ShelfError`, never `Response` objects
- Use `try/catch` blocks with meaningful error messages
- Include `additionalData` for debugging context

**Key utilities:** `ShelfError`, `makeShelfError()`, `payload()`, `error()`, `parseData()`

See [docs/handling-errors.md](./docs/handling-errors.md) for complete documentation.

### Bulk Operations & Select All Pattern

When implementing bulk operations that work across multiple pages of filtered data, follow the **ALL_SELECTED_KEY pattern**:

**The Pattern:**

1. **Component Layer** - Pass current search params when "select all" is active
2. **Route/API Layer** - Extract and forward `currentSearchParams`
3. **Service Layer** - Use `getAssetsWhereInput` helper to build where clause from params

**Key Implementation Points:**

- Use `isSelectingAllItems()` from `app/utils/list.ts` to detect select all
- Always pass `currentSearchParams` alongside `assetIds` when ALL_SELECTED_KEY is present
- Use `getAssetsWhereInput({ organizationId, currentSearchParams })` to build Prisma where clause
- Set `takeAll: true` to remove pagination limits

**Working Examples:**

- Export assets: `app/components/assets/assets-index/export-assets-button.tsx`
- Bulk delete: `app/routes/_layout+/assets._index.tsx` (action)
- QR download: `app/routes/api+/assets.get-assets-for-bulk-qr-download.ts`

**📖 Full Documentation:** See [docs/select-all-pattern.md](./docs/select-all-pattern.md) for detailed implementation guide, code examples, and common pitfalls.

## Testing Approach

### Unit Tests (Vitest)

- Tests co-located with source files (e.g., `service.server.test.ts` next to `service.server.ts`)
- Happy DOM environment for React component testing
- Run with `npm run test` or `npm run test:cov` for coverage

### E2E Tests (Playwright)

- Located in `test/e2e/` directory
- Requires app running on localhost:3000
- Run `npm run test:e2e:install` first to install browsers

### Validation Pipeline

Always run `npm run validate` before committing - this runs tests, linting, and typecheck in parallel.

### Writing & Organizing Tests

#### Test Philosophy

- Write behavior-driven tests focusing on observable outcomes rather than implementation details.
- Tests should describe what the system does, not how it does it.
- Avoid testing internal private methods or state; instead, test public interfaces and user-visible effects.

#### When to Mock

- Mock only external network calls, time-based functions, feature flags, or heavy dependencies that are impractical or slow to run in tests.
- Avoid mocking internal business logic or utility functions to keep tests realistic and maintainable.
- Prefer using real implementations where possible to catch integration issues early.

#### Mock Justification Rule

- Every mock must be accompanied by a `// why:` comment explaining the reason for mocking.
- This encourages thoughtful use of mocks and helps reviewers understand test design choices.

#### Organizing Mocks and Factories

- **Test files**: Co-located with source files (e.g., `app/modules/user/service.server.test.ts`)
- **Shared mocks**: Place in `test/mocks/` directory, organized by domain (remix.tsx, database.ts)
- **Factories**: Place in `test/factories/` directory for generating test data
- **MSW handlers**: Keep in root `mocks/` directory for API mocking

Example directory structure:

```
app/
├── modules/
│   └── user/
│       ├── service.server.ts
│       └── service.server.test.ts  # Co-located test
test/
├── mocks/
│   ├── remix.tsx          # Remix hook mocks
│   └── database.ts        # Database/Prisma mocks
└── factories/
    ├── user.ts            # User factory
    ├── asset.ts           # Asset factory
    └── index.ts           # Export all
mocks/                      # MSW API handlers (kept at root)
├── handlers.ts
└── index.ts
```

#### Path Aliases

Path aliases are configured via `tsconfig.json` and resolved by `vite-tsconfig-paths`:

```typescript
import { something } from "~/utils/something"; // → app/utils/something.ts
```

#### Factories & Test Data

- Use factories to generate consistent and realistic test data.
- Factories should allow overrides for specific fields to tailor data for each test case.
- Avoid hardcoding data within tests; use factories to keep tests clean and maintainable.

Example factory usage:

```typescript
import { faker } from "@faker-js/faker";

const testUser = {
  id: faker.string.uuid(),
  email: faker.internet.email(),
  firstName: faker.person.firstName(),
};
```

## Environment Configuration

### Required Environment Variables

- `DATABASE_URL` and `DIRECT_URL` - PostgreSQL connections
- `SUPABASE_URL` and `SUPABASE_ANON_PUBLIC` - Supabase configuration
- `SESSION_SECRET` - Session encryption key

### Feature Flags

- `ENABLE_PREMIUM_FEATURES` - Toggle subscription requirements
- `DISABLE_SIGNUP` - Control user registration
- `SEND_ONBOARDING_EMAIL` - Control onboarding emails

## Important Files to Understand

1. **`app/database/schema.prisma`** - Complete database schema and relationships
2. **`app/config/shelf.config.ts`** - Application configuration and constants
3. **`app/modules/`** - Core business logic services (asset, booking, user, etc.)
4. **`app/routes/_layout+/`** - Main authenticated application routes
5. **`vite.config.ts`** - Build configuration with Remix and development settings

## Development Workflow

1. **Database Changes**: Modify `schema.prisma` → `npm run db:prepare-migration` → `npm run db:deploy`
2. **New Features**: Create in `app/modules/` for business logic, `app/routes/` for pages
3. **Component Updates**: Follow existing patterns in `app/components/`
4. **Testing**: Write unit tests for utilities  
   Follow the testing conventions outlined in the Writing & Organizing Tests section to ensure consistent, behavior-driven testing and minimal mocking.
5. **Pre-commit**: Always run `npm run validate` to ensure code quality

## Git and Version control

- add and commit automatically whenever a task is finished
- Always use Conventional Commits spec when making commits and opening PRs: https://www.conventionalcommits.org/en/v1.0.0/
- use descriptive commit messages that capture the full scope of the changes
- **IMPORTANT: Commit message lines must be ≤ 100 characters**
  - Both subject line (header) and body lines are limited to 100 characters
  - Wrap long lines to stay within the limit
  - This is enforced by commitlint pre-commit hook
- dont add 🤖 Generated with [Claude Code](https://claude.ai code) & Co-Authored-By: Claude <noreply@anthropic.com>" because it clutters the commits
- Include test readability and mock discipline in PR reviews. Overly mocked or verbose tests should be refactored before merge.

## Maintaining This Document

Update CLAUDE.md when:
- New patterns are used consistently across 3+ files
- Common bugs could be prevented by documented rules
- Code reviews repeatedly mention the same feedback
- Libraries or architectural patterns change

Keep rules actionable, examples from actual code, and references up to date.
