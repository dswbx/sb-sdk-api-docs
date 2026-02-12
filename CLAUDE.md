# Repository Overview

This is a documentation repository for Supabase-JS API reference. Goal: document all public-facing APIs to support building lightweight, compatible backends (SQLite-based for Database APIs, TypeScript-based for Auth APIs).

**Not mission-critical** - building compatible system with only needed features, not production-ready GoTrue/PostgREST replacement.

# Project Structure

```
sdk/
├── database-api-plan.md          # Plan: Database APIs (postgrest-js)
├── auth-api-plan.md              # Plan: Auth APIs (auth-js/GoTrue)
├── database/
│   ├── README.md                 # Navigation guide
│   ├── progress.md               # Track completion status
│   └── [16 content files]        # Method documentation
└── auth/
    ├── README.md
    ├── progress.md
    └── [content files per plan]
```

# Documentation Categories

1. **Database APIs** (~66 methods) - ✅ COMPLETE → see `sdk/database/progress.md`
2. **Auth APIs** (~70 methods) - 🔄 IN PROGRESS → see `sdk/auth/progress.md`
3. **Storage APIs** (~25 methods) - TODO
4. **Realtime APIs** (~50 methods) - TODO
5. **Functions APIs** (~2 methods) - TODO

# Architecture & Approach

## Three-Tier Priority System

All API documentation uses standardized priority tiers:

- **HIGH Priority** (~80% usage) - Core methods, full detail documentation
- **MEDIUM Priority** (~15% usage) - Advanced features, moderate detail
- **LOW Priority** (~5% usage) - Specialized/enterprise, brief format only

## Documentation Format Standards

**HIGH/MEDIUM methods** include:

- Full signature with TypeScript types
- Purpose, parameters, return types
- Usage examples (basic + advanced)
- Implementation considerations (backend requirements, security, complexity)
- Flow diagrams for complex operations
- Error cases
- Related methods

**LOW methods** use brief format:

- One-line purpose
- Basic signature
- Single usage example
- Implementation note
- Use case

## Plan Files Structure

Each `*-api-plan.md` at `sdk/` root contains:

- Scope, output structure, priority breakdown
- Documentation format examples, workflow phases
- Source file references, progress checklist
- Unresolved questions (answered after implementation starts)

## Category-Specific Analysis

**Database APIs:**

- Analysis focus: SQLite compatibility (✅ Full / ⚠️ Partial / ❌ No support)
- Includes compatibility matrix with implementation roadmap
- Source: `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/`

**Auth APIs:**

- Analysis focus: Implementation complexity (✅ Simple / ⚠️ Moderate / ❌ Complex)
- Includes security considerations, backend requirements, authentication flows
- Building GoTrue-compatible backend in TypeScript (non-mission-critical)
- OAuth scope: Major providers only (Google, GitHub, Apple)
- MFA/WebAuthn/SSO: Deferred to LOW priority
- Admin API: Server-side only
- Source: `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/`

# Key Decisions (Auth)

From `auth-api-plan.md` unresolved questions (answered):

1. **Target:** GoTrue-compatible backend, TypeScript, non-mission-critical
2. **OAuth:** Major providers only (Google, GitHub, Apple)
3. **MFA:** LOW priority (deferred)
4. **Admin API:** Server-side only
5. **WebAuthn:** LOW priority (deferred)
6. **SSO/SAML:** LOW priority (deferred)

# Working with This Repository

## Using Progress Files to Continue Work

Each category has `progress.md` tracking completion status. To continue:

1. Open relevant `progress.md` (e.g., `sdk/auth/progress.md`)
2. Find first unchecked item or file marked 🔄 IN PROGRESS
3. Read corresponding `*-api-plan.md` for format/priority details
4. Document methods per plan format
5. Update `progress.md` checkbox after completing file
6. Mark file status: ✅ COMPLETE when done

## Implementing Documentation Plans

When documenting API category:

1. Read `*-api-plan.md` completely
2. Follow workflow phases sequentially
3. Use exact documentation format from plan
4. Update `progress.md` after each file/section
5. Mark plan checklist items complete
6. Create reference docs last (api-overview, compatibility/complexity matrix)

## Source Code References

Source files are in adjacent repo:

- Database: `/Users/dennis/Projects/supabase-js/packages/core/postgrest-js/src/`
- Auth: `/Users/dennis/Projects/supabase-js/packages/core/auth-js/src/`

External docs:

- PostgREST: https://docs.postgrest.org/en/v14/index.html

Read source to extract:

- Method signatures (with generics)
- JSDoc comments
- Type definitions
- Usage patterns from tests

## Maintaining Consistency

- File naming: lowercase with hyphens (e.g., `core-authentication.md`)
- Output location in plan: Use absolute paths for clarity
- Method counts in brackets: `[X/Y]` format (completed/total)
- Progress format: Checkboxes with method counts
- Status indicators: ✅ Complete, 🔄 In Progress, ❌ Blocked, blank = TODO
- Concise writing: Sacrifice grammar for brevity per user's global instructions

## Updating This File (CLAUDE.md)

Update CLAUDE.md when discussions/work reveal new insights:

**What to update:**

- Key decisions made during implementation (add to relevant category section)
- New architectural patterns discovered (add to Architecture section)
- Scope changes or priority adjustments (update category-specific sections)
- Structural changes to project layout (update Project Structure)
- Workflow improvements (update Working with This Repository)

**When to update:**

- After resolving unresolved questions from plan files
- When establishing new conventions or standards
- After completing major category work (update status indicators)
- When discovering better approaches during implementation

**How to update:**

- Keep existing structure, add/modify relevant sections only
- Maintain concise format per global instructions
- Update status indicators in Documentation Categories section
- Add new Key Decisions sections for other categories as needed
- Preserve examples and format standards
