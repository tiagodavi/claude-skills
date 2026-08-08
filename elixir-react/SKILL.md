---
name: elixir-react
description: guidelines for elixir/react projects
license: MIT
metadata:
 author: Tiago Davi
 version: 1.0.0
 category: productivity
 tags: [elixir, phoenix, liveview, react, apollo, graphql]
 support: tiago.asp.net@gmail.com
---

# General 

1. **Comments:** * Only add if really needed and always keep them short, concise and right to the point, don't be verbose.

2. **Bug Prevention:** Avoid introducing regressions while modifying the codebase, especially subtle or hard-to-detect bugs. Ensure features are properly tested before completion.

3. **Security:** Always prioritize system security. Never expose credentials, weaken protections, or introduce vulnerabilities that could compromise the platform.

4. **Debuggability:** Add structured debug logs tagged with the feature name and metadata covering important control flows, edge cases, and failure scenarios to simplify troubleshooting and observability.

5. **Divide and Conquer:** Break the work into incremental steps and implement one module at a time. Request human review after each major step before proceeding. Maintain clear tracking of pending tasks and continue iterating until the full implementation is complete without leaving unfinished work behind.

# Elixir / Phoenix / Absinthe (GraphQL)

## General Guidelines:

* Always try to use standard mix generators instead of manually generating files mainly for database migration files. 

* Always run `mix format` at the end of each implementation to ensure the codebase remains consistently formatted.

* Always run `MIX_ENV=test mix test` at the end. If the test environment is corrupted or inconsistent, try:
  `MIX_ENV=test mix ecto.drop && MIX_ENV=test mix ecto.setup`

1. **Imports / Requires / Aliases:** Keep all `import`, `require`, and `alias` statements at the top of the file for consistency and readability.

2. **TDD First:** Test-Driven Development is mandatory. For every feature:

   * Write at least 3 happy-path tests
   * Write at least 3 failure-path tests
   * Write at least 3 edge-case tests

   Tests should fail first, then the feature should be implemented until all tests pass.

3. **BDD Style:** Prefer behavior-driven tests whenever possible. Example:
   `Given user X on screen Y, action Z should happen.`

4. **Type Specifications:** Document modules and public functions thoroughly. Add `@spec` annotations and custom types whenever possible to improve maintainability and tooling support.

5. **Explicit Over Implicit:** Code should be self-explanatory, easy to reason about, and optimized for readability over cleverness.

6. **Jobs Organization:** Any background job implementation should live under:
   `lib/<app>/jobs`

7. **Services Organization:** Any integration with external services, APIs, or providers should live under:
   `lib/<app>/services`

8. **Schemas Organization:** Ecto schemas, database models, and schema-related modules should live under:
   `lib/<app>/schemas`

9. **Protocols:** Use protocols to avoid duplicated conditional logic when behavior changes depending on data structure or type.

10. **Behaviours:** Use behaviours whenever modules must conform to a shared contract or interchangeable implementation pattern.

11. **GraphQL Performance:** Always look for and eliminate N+1 query issues using DataLoader whenever applicable.

12. **Readable Ecto Queries:** Prefer named bindings in Ecto queries to improve readability and maintainability.

13. **Transactions:** Use transactions whenever operations must rollback atomically on failure. However, evaluate carefully whether the entire workflow truly needs to execute inside a blocking transaction.

14. **Migration Safety & Performance:** Design migrations to execute as efficiently as possible. Ensure indexes are aligned with query patterns and operational needs. If a migration may lock tables or run for too long, consider offloading the work to background jobs or phased migrations.

15. **Concurrency & IO:** Leverage Elixir’s concurrency model to parallelize IO-bound workloads whenever possible, while keeping execution flow understandable and maintainable.

## Anti-patterns

1. Read this before creating code to make sure you don't introduce anti-patterns: `./references/elixir-anti-patterns-reference.md`

# TypeScript / React / Apollo (GraphQL)

## General Guidelines:

The full UI codebase is located under the `ui` folder.

Before implementing any UI-related code, always review:
`./references/solid-react-reference.md`

This document defines the architectural and implementation guidelines that must be followed consistently across the frontend codebase.

1. **UI Contract Awareness:** Always inspect the existing React/Apollo implementation and GraphQL interactions before making changes to ensure the full request/response contract and data flow are clearly understood.

2. **Caching & UX:** Always evaluate opportunities to improve caching behavior, query efficiency, and client-side state management to enhance responsiveness and overall user experience.

3. **UI Testing:** Always add tests covering new UI functionality. Follow the same BDD-style testing philosophy used in the Elixir backend to ensure consistent behavior validation across the stack.
