## Purpose

Redis-backed `PubSubEngine` for `graphql-subscriptions`, so multiple GraphQL servers can share subscription events over Redis pub/sub.

## Project Snapshot

- Type: single npm package (`@lvt/graphql-redis-subscriptions`)
- Tech: TypeScript → CommonJS (`tsc`), peer `graphql-subscriptions`, optional `ioredis`
- Entry: `src/index.ts` exports `RedisPubSub` / `PubSubRedisOptions`; `main` is `dist/index.js`
- Publish: `publishConfig.registry` is LVT JFrog; `.github/workflows/publish.yml` targets npmjs.org

## Commands

```bash
npm ci
npm run compile          # tsc → dist/
npm run watch            # tsc -w
npm test                 # coverage + lint (needs Redis)
npm run testonly         # unit tests (mocked Redis client)
npm run integration      # live Redis + cluster tests
npm run coverage         # nyc over src/test/**/*.ts
npm run lint             # eslint src --ext ts
```

**Redis required for `npm test` / `coverage` / `integration`:** local `redis:alpine` on `6379`, and for cluster tests `grokzen/redis-cluster` on ports `7001–7006` (see README and `.github/workflows/test.yml`).

## Conventions

- Branch/PR: GitHub Actions test on `master` push and PRs; publish on GitHub Release via `.github/workflows/publish.yml`
- Public API surface is what `src/index.ts` exports — do not assume other `src/*.ts` modules are package exports
- Prefer injecting `publisher` / `subscriber` Redis clients in production (see README); default constructor auto-`require`s `ioredis`
- Do not pass both `reviver` and `deserializer` to `RedisPubSub` — constructor throws (`src/redis-pubsub.ts`)

## Directory Map

- `src/` — library implementation (`redis-pubsub.ts`, `pubsub-async-iterator.ts`, `with-filter.ts`)
- `src/test/` — mocha unit (`tests.ts`) and integration (`integration-tests.ts`) suites
- `.github/workflows/` — `test.yml` (Redis services + `npm test`), `publish.yml` (release publish)
- `dist/` — build output (gitignored; produced by `npm run compile` / `prepublish`)

## Architecture

1. `RedisPubSub.publish` → Redis `PUBLISH` (JSON / Buffer / custom serializer)
2. `RedisPubSub.subscribe` → Redis `SUBSCRIBE` or `PSUBSCRIBE` when `options.pattern`
3. Concurrent same-trigger subscribes coalesce via `subsPendingRefsMap` (`src/redis-pubsub.ts`)
4. `asyncIterator` / `asyncIterableIterator` return `PubSubAsyncIterator` (`src/pubsub-async-iterator.ts`) for GraphQL `subscribe` resolvers

## Patterns

- DO inject paired `publisher` + `subscriber` clients for prod retry/cluster control — `src/redis-pubsub.ts`, README
- DO use `options.pattern: true` for Redis pattern channels — `src/redis-pubsub.ts`, `src/test/integration-tests.ts`
- DON'T call `unsubscribe` expecting `punsubscribe` for pattern subs — `unsubscribe` only calls `redisSubscriber.unsubscribe` (`src/redis-pubsub.ts`); JET-31291 removed `punsubscribe`
- DON'T import `withFilter` from this package — not exported from `src/index.ts`; tests use `src/with-filter.ts`, README uses `graphql-subscriptions`

## Key Files

- `src/index.ts` — public exports
- `src/redis-pubsub.ts` — `RedisPubSub` engine
- `src/pubsub-async-iterator.ts` — AsyncIterator bridge
- `src/test/tests.ts` — unit tests with mocked Redis
- `src/test/integration-tests.ts` — GraphQL subscribe + Redis Cluster

## Gotchas

- **Publish target unclear:** `package.json` `publishConfig` → JFrog; Actions publish → npmjs — confirm before releasing
- **Docs lag fork:** README/install still describe unscoped `graphql-redis-subscriptions`; package name is `@lvt/graphql-redis-subscriptions`
- **homepage/bugs** in `package.json` still point at upstream `davidyaha/graphql-redis-subscriptions`
