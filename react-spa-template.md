# React SPA Template

A repeatable starting point for new frontends, modelled on `lawbrokr-frontend`. It replaces
`npx create-next-app@latest .` with a scaffold plus a fixed set of conventions: four source layers with
enforced import boundaries, one feature folder per product domain, a token-only design system, and a single
quality-gate command.

The stack is React 19, Vite, TypeScript, Tailwind 4, TanStack Router and TanStack Query. No server
rendering, no database access. It assumes a separate API.

---

## 1. Bootstrap a new repo

There is no single official scaffolder for this exact shape, so it is two steps: generate the router
skeleton, then apply the conventions below.

```bash
mkdir my-app && cd my-app
pnpm dlx create-tsrouter-app@latest . --template file-router --tailwind --package-manager pnpm
```

`create-tsrouter-app` is TanStack's own builder. The `file-router` template gives file-based routing and
the Vite router plugin already wired. It does not give the layer structure, the boundary rules, the quality
gate or the design-token setup. Those are sections 3 onward.

Alternatives, if the scaffolder ever changes: `pnpm create vite . --template react-ts` and add the router
plugin by hand, or copy an existing repo and strip its `src/features`.

**Make step 1 a one-liner for yourself.** Once you have gone through this document once, the finished repo
is the template. Push it to GitHub, mark it a template repository, and every later project starts with:

```bash
gh repo create my-app --template <you>/react-spa-template --private --clone
```

That is the real answer to "something as easy as create-next-app". The scaffolder gets you a router. Your
own template repo gets you this whole document already applied.

---

## 2. Machine prerequisites

| Tool     | Why                                                                              |
| -------- | -------------------------------------------------------------------------------- |
| fnm      | Reads `.nvmrc` so developers and CI run the same Node major                      |
| pnpm 11  | Pinned by `packageManager` in `package.json`; never npm or yarn                  |
| direnv   | Exports `.envrc` into every shell in the repo, so tools inherit env vars          |
| lefthook | Git hooks: lint and format staged files, refuse secret files, enforce commit form |

`.nvmrc` holds only the major, for example `24`. It is the single source for the Node version. When the
next Active LTS lands, the move is one edit to that file.

---

## 3. The four layers

This is the core of the template. Everything else supports it.

```
src/app/        bootstrap: main.tsx, providers, validated env, error reporting, mock switch
src/routes/     file-based routes — thin: guards + compose features
src/features/   one folder per product domain; public API is its index.ts
src/shared/     api · ui · lib · config · styles · brand · test
src/types/      ambient declarations
test/unit/      Vitest, mirrors src/
test/e2e/       Playwright specs
docs/           architecture · testing · tooling · decisions/ (ADRs)
```

**The import direction is the rule that makes it hold:**

- `shared` may import only `shared`.
- `features` may import `shared`, and other features **only through their `index.ts`**.
- `app` and `routes` may import anything.

Two tools enforce it, deliberately overlapping. `eslint-plugin-boundaries` reports at the import site while
you type. `dependency-cruiser` catches cycles and orphans the linter cannot see. If either blocks an
import, the design is wrong, not the config.

### What goes in a feature folder

```
src/features/<domain>/
  api/         TanStack Query options, mutations, MSW mocks for this domain
  components/  the domain's React components
  model/       types, Zod schemas, URL search schemas
  lib/         pure helpers (formatting, display logic)
  store/       Zustand slice, only if the domain needs client state
  index.ts     the public API — the only file another feature may import
```

Keep `index.ts` deliberately small. It is a contract, not a barrel file. Export the components, query
options and schemas another layer actually needs, and nothing else.

### Route files stay thin

A route runs a guard, validates its search params, and composes components from a feature. It holds no
business logic.

```tsx
export const Route = createFileRoute('/_app/_customer/leads/')({
  beforeLoad: requireFeature('leads'),
  validateSearch: zodValidator(leadsSearchSchema),
  search: { middlewares: [stripSearchParams(DEFAULT_SEARCH)] },
  component: LeadsPage,
});
```

Guards live in `src/routes/-guards.ts`. The leading dash keeps the file out of the generated route tree.
Two guards cover most apps: one redirects a session in the wrong surface to its own home, one returns a
**404** for a feature the permission manifest does not grant. Return 404, never 403. Confirming a page
exists is itself information.

### Route grouping for multiple audiences

If one app serves several kinds of user, use pathless layout routes rather than separate apps:

```
src/routes/
  __root.tsx                 router context: queryClient + session store
  _app.tsx                   auth gate + the signed-in chrome
  _app/_customer.tsx         surface guard
  _app/_customer/*.tsx       customer pages at root URLs
  _app/_agency/*.tsx         agency pages under /agency
  _app/_platform/*.tsx       admin pages under /admin
  (public)/login.tsx         unauthenticated pages
  -guards.ts                 shared beforeLoad helpers
```

Which tree a user lands in comes from the session context the API returns. The navigation shows only what
their permission manifest grants.

---

## 4. Data layer

**Generate the API client. Never hand-write it.** Point `@hey-api/openapi-ts` at the backend's OpenAPI
document and generate types, the SDK, TanStack Query options, Zod schemas and typed MSW handler factories
in one pass.

```ts
// openapi-ts.config.ts
export default defineConfig({
  input: './openapi/openapi.json',
  output: 'src/shared/api/generated',
  plugins: ['@hey-api/typescript', '@hey-api/sdk', '@tanstack/react-query', 'zod', 'msw'],
});
```

`src/shared/api/generated/` is generated and never edited. So is `src/routeTree.gen.ts`. Both are excluded
from ESLint, Prettier, coverage and the duplication check.

Around the generated client sits a thin hand-written wrapper: `client.ts` for configuration, `errors.ts`,
`pagination.ts`, `query-client.ts`, and `sse.ts` if you need server-sent events.

**Which tool holds which state:**

| State                     | Tool                          |
| ------------------------- | ----------------------------- |
| Server data               | TanStack Query                |
| Session and identity      | Zustand store                 |
| Filters, sort, pagination | The URL, via a Zod search schema |
| Form state                | react-hook-form + Zod resolver |

Putting filters in the URL is worth calling out. It makes every list view linkable and restores itself on
reload for free.

**Mock the backend from day one.** MSW handlers per domain in `features/<domain>/api/*.mocks.ts`, assembled
in `src/app/mocks.ts` because app is the one layer allowed to import every feature. Gate it on a raw
literal so the bundler proves the branch dead and drops MSW entirely from a production build:

```ts
if (import.meta.env.VITE_API_MOCK === '1') {
  const { startMocks } = await import('./mocks');
  await startMocks();
}
```

The payoff is that `pnpm dev` works with no backend running, and every test uses the same fake.

**Reach a real API same-origin, at `/api/*`.** Vite's dev and preview servers proxy that prefix; the host
rewrites it in production. This matters if the session uses a CSRF cookie the page has to read, since a
page reads only its own origin's cookies.

---

## 5. Environment

Validate the environment once at boot with Zod, in `src/app/env.ts`, and export a typed object. A malformed
value then fails at startup with a readable message instead of deep inside a request.

```ts
const schema = z.object({
  VITE_API_URL: z.union([z.literal(''), z.url()]).default(''),
  VITE_API_MOCK: z.enum(['', '0', '1']).default(''),
  VITE_SENTRY_DSN: z.string().default(''),
});

const parsed = schema.safeParse(import.meta.env);
if (!parsed.success) throw new Error(`Invalid environment:\n${z.prettifyError(parsed.error)}`);
```

Everything prefixed `VITE_` is inlined into the bundle and is therefore **public**. Never put a secret
there. Build-time-only secrets, such as a source-map upload token, go in unprefixed variables.

Keep one committed `.envrc.example` listing every variable with a comment, and one gitignored `.envrc`
holding the values. direnv exports it into every shell opened in the repo.

---

## 6. Design system

The single rule: **colour, radius, shadow, font, tracking and motion values live in exactly one file.**

```
src/shared/styles/
  tokens.css    the only file allowed to hold raw values
  theme.css     maps tokens onto Tailwind's theme variables; holds no values itself
  global.css    imports tailwind, the animation utilities, then the two files above
```

In `theme.css`, clear Tailwind's default palette:

```css
@theme inline {
  --color-*: initial;
  --color-background: var(--background);
  --color-primary: var(--primary);
  /* …every semantic role, 1:1 with tokens.css */
}
```

After that, `text-white`, `bg-slate-100` and `bg-black/50` compile to nothing. A colour that is not a token
cannot reach a component by accident. This one line prevents more drift than any review process.

**If the app is light-only, say so structurally.** Re-point the dark variant at a class nothing sets, so the
`dark:` utilities inside generated shadcn primitives stay inert rather than following the operating system:

```css
@custom-variant dark (&:where(.dark, .dark *));
```

### The three UI tiers

```
src/shared/ui/
  *.tsx        primitives from the shadcn CLI, restyled through tokens and cva
  blocks/      composed layout: app-shell, sidebar, page-header, stat-card, empty-state, confirm-dialog
  charts/      chart wrappers plus a chart-theme.ts that binds them to tokens
  grid/        data-grid wrapper plus grid-theme.ts
  kit/         catalogue entries
```

**Build the `/__kit` route early.** It renders every component in every variant on one page. It is where you
review a UI change, and it is the surface the accessibility and viewport tests run against. Gate it so it
is always on in development and opt-in for a build, then a production deploy answers "page not found"
unless you deliberately enable it for a reviewer.

Point `components.json` at your aliases so the shadcn CLI writes into `src/shared/ui`:

```json
{
  "tailwind": { "css": "src/shared/styles/global.css", "cssVariables": true },
  "aliases": { "ui": "@/shared/ui", "utils": "@/shared/lib/utils" },
  "iconLibrary": "lucide"
}
```

---

## 7. The quality gate

One command, chained so the first failure stops it.

```json
"check": "pnpm typecheck && pnpm lint && pnpm format:check && pnpm knip && pnpm dup && pnpm depcruise && pnpm test:coverage"
```

| Step         | Tool               | Catches                               |
| ------------ | ------------------ | ------------------------------------- |
| typecheck    | `tsc --noEmit`     | Types, at strictest settings          |
| lint         | ESLint             | `--max-warnings 0`; no warning budget |
| format:check | Prettier           | Formatting drift                      |
| knip         | knip               | Dead files, exports and dependencies  |
| dup          | jscpd              | Copy-paste, at a 0% threshold         |
| depcruise    | dependency-cruiser | Cycles, orphans, layer violations     |
| test:coverage| Vitest             | Unit tests with enforced thresholds   |

`pnpm audit` runs separately, as its own CI job. It needs the registry, so a registry outage fails visibly
on its own rather than masking a lint result.

**The rule that makes it work: zero errors and zero warnings.** A warning genuinely outside your control is
fixed by narrowing the configuration, never by tolerating it in the output. Once one warning is acceptable,
the gate stops meaning anything.

### TypeScript

```json
{
  "extends": ["@tsconfig/vite-react/tsconfig.json", "@tsconfig/strictest/tsconfig.json"],
  "compilerOptions": {
    "noEmit": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "test", "*.config.ts"]
}
```

Start at `@tsconfig/strictest` and relax individual flags with a comment if something proves impractical.
Starting loose and tightening later never happens.

### Testing split

Vitest covers logic. Playwright covers rendering. Do not blur the line.

Coverage is measured only over the logic layers: `shared/api`, `shared/lib`, `shared/config`, and each
feature's `api`, `lib`, `model` and `store`. Components are excluded on purpose. Unit-rendering a primitive
to move a percentage adds near-duplicate tests and guarantees nothing.

Playwright runs three viewports, for example 1920, 1440 and 414, with an axe accessibility pass. Consider
keeping it out of CI if the suite grows large, and enforce it locally before a change is called done.
That is a real trade-off, so write down which way you chose and why.

### Git hooks

`pre-commit` runs ESLint `--fix` and Prettier over staged files and restages them. It also **refuses a
commit that stages a secret file** — `.envrc`, `.env`, `.env.*`, anything under `.secrets/` — while letting
the committed `.envrc.example` through. `commit-msg` enforces Conventional Commits.

The secret guard matters more than it looks. `.gitignore` alone does not stop a force-add.

---

## 8. Documentation that belongs in the repo

- `README.md` — machine setup, the command table, how to sign in locally, how to deploy.
- `docs/architecture.md` — the layers, and honestly which parts are real versus scaffolding.
- `docs/testing.md` — what is unit-tested, what is end-to-end, and why coverage excludes components.
- `docs/tooling.md` — why each tool is configured the way it is, and what breaks when a step is skipped.
- `docs/decisions/` — numbered ADRs.

The ADRs are the highest-value part and the easiest to skip. One short file per irreversible decision:
the stack versions, the API client approach, the auth and session model, the data patterns, where the
design source of truth lives, and the palette-clearing choice. Each records what was decided, what was
rejected, and why. Six months later this is the only thing that answers "why is it like this".

**Seed a `CLAUDE.md` too**, holding the layer rules, the conventions, and the workflow you want an agent
to follow. The import boundaries and the quality gate are exactly the kind of constraint an agent
otherwise has to rediscover by tripping over the linter.

---

## 9. Setup checklist for a new repo

1. Scaffold with `create-tsrouter-app`, or clone your own template repo and skip to step 9.
2. Write `.nvmrc`, set `packageManager` to the pinned pnpm version, add `engines.node`.
3. Create the four source layers, plus `test/unit`, `test/e2e` and `docs/decisions`.
4. Add `eslint.config.ts` with type-checked rules, the boundaries plugin and a11y, then
   `.dependency-cruiser.cjs` with the same layer policy.
5. Add `tokens.css`, `theme.css` and `global.css`, clearing the Tailwind palette. Point
   `components.json` at `src/shared/ui`.
6. Set up the API: OpenAPI generation, the client wrapper, the query client, and MSW per domain.
7. Validate the environment in `src/app/env.ts`. Commit `.envrc.example`, gitignore `.envrc`.
8. Wire the gate: the `check` script, `lefthook.yml` with the secret guard, and a CI workflow that runs
   `check`, `build` and `audit`.
9. Build the `/__kit` route and the app shell before the first real screen.
10. Write ADR 0001 and 0002 the same day. They are never easier to write than now.

---

## 10. Where the judgement calls actually are

Most of this is mechanical. Four decisions are not, and each deserves a sentence in an ADR:

- **How much the route layer is allowed to know.** Keeping it to guards plus composition is what stops
  routes becoming the place logic lands by default.
- **Whether features may import each other at all.** Through `index.ts` only is a good middle. Fully
  isolated pushes everything into `shared`, which then becomes the dumping ground you were avoiding.
- **What coverage measures.** Excluding components is defensible and worth stating, because the number
  looks lower and someone will eventually ask.
- **Whether end-to-end tests run in CI.** Out of CI means faster pipelines and a discipline that depends on
  people. In CI means the opposite. Either is fine. Being vague about it is not.
