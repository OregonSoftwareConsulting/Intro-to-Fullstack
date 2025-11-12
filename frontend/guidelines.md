# Frontend Guidelines (React + Next.js)

Opinionated basics for new contributors. Emphasis on opinionated, these are good guidelines to get you started but you will discover patterns that work best for you as you develop. Also, deadlines usually mean guidelines go out the window. A perfectly planned and executed project is near impossible, so that is sometimes just the way it is.

---

## 1) Composition over Inheritance

* Prefer **small, focused components** that you compose together.
* Extract behavior into **hooks** and UI into presentational components.
* Apply the same design principles (**composition, reusability, and no repetition**) to **hooks** as to components.

```tsx
// Bad: monolithic "SmartButton"
function SmartButton() { /* fetch + logic + styles + markup */ }

// Good: compose behavior + UI
function useSaveThing() { /* state + side effects */ }

// You would probably want a little more than this to justify a separate component. However, you might also try thinking ahead and realize you may want a spinner for all of your save buttons later on while pending. Even if this minimal implementation seems overkill right now, this saves future development time.
function SaveButton(props: { onClick: () => void; disabled?: boolean }) {
  return <button disabled={props.disabled} onClick={props.onClick}>Save</button>;
}

export default function Toolbar() {
  const { mutate, isPending } = useSaveThing();
  return <SaveButton onClick={mutate} disabled={isPending} />;
}
```

**Heuristics**

* If a prop is boolean and changes rendering: consider a new child component.
* If a file exceeds ~150–200 lines or mixes data + layout + styles: split it.
* **Hooks** should remain single-purpose, composable, and self-contained.

---

## 2) Avoid Repetition via Reuse & Abstractions

* Create **reusable UI primitives** (Button, Input, Modal) and **layout shells** (Page, Section).
* Factor **data fetching or business logic** into hooks: `useUser()`, `useProjects()`.
* Share **utility functions** (formatting, date math, URL builders) — many **utils are good candidates to become hooks** if they depend on React state or lifecycle.

```tsx
// ui/Button.tsx
export function Button({ as: Comp = 'button', ...props }) {
  return <Comp className="btn" {...props} />;
}

// hooks/useProject.ts
export function useProject(id: string) { /* shared fetching/caching */ }

// utils/date.ts
export function formatDate(d: string) { /* ... */ }
```

**Guideline**

* Hooks follow the same rules as components: keep them small, composable, and DRY.
* Avoid repeating logic across hooks—compose them instead.

---

## 3) Rendering Modes in Next.js (SSR/SSG/ISR)

* **SSR (Server-Side Rendering)**: dynamic, per-request HTML. Great for personalized or frequently changing data that must be fresh for SEO.

  * Use `async` **Server Components** or **Route Handlers**.
* **SSG (Static Site Generation)**: build-time HTML. Best for mostly static content.
* **ISR (Incremental Static Regeneration)**: static with periodic revalidation (`revalidate`), balancing speed + freshness.
* **Client Components**: only when you need interactivity (events, state, browser APIs).

**Rules of thumb**

* Default to **Server Components** for data loading → smaller bundles, better TTFB/SEO.
* Promote **progressive enhancement**: render core content on the server, hydrate interactivity where needed.
* For client-side data that must be live (dashboards), combine SSR shell + client queries.

---

## 4) Semantics & HTML First

* Use semantic tags: `header`, `main`, `nav`, `section`, `article`, `aside`, `footer`.
* Preserve heading order (`h1` → `h2` → `h3`…).
* Inputs must have **explicit labels**; group related inputs with `fieldset/legend`.
* Use **button** for actions, **a** for navigation (with `next/link` for internal navigation).

```tsx
<main>
  <h1>Account</h1>
  <section aria-labelledby="security">
    <h2 id="security">Security</h2>
    ...
  </section>
</main>
```

---

## 5) Accessibility (a11y) Essentials

* Firstly, why should you care? There are lawyers who's sole purpose is to find and sue sites that are not in compliance with ADA. ADA is the law that basically says don't discriminate against disabled people. WCAG is a framework used to ensure you are following accessibility best practices.
*  **Keyboard first**: focus states visible, logical tab order.
* Provide **aria-label/aria-labelledby** only when semantics aren’t sufficient.
* **Color contrast** ≥ WCAG AA. Don’t rely on color alone.
* **Live regions** (`aria-live="polite"`) for background updates.
* Manage focus on dialogs, toasts, route changes.
* Prefer well-tested components (e.g., Radix UI/shadcn) that respect a11y.

---

## 6) State: Controlled vs. Uncontrolled (Prefer Controlled)

* **Controlled components** keep form state in React; they’re predictable and easier to validate, test, and sync.
* **Uncontrolled** (refs, native form state) can be fine for simple, non-critical inputs but are harder to coordinate.

```tsx
// Controlled (preferred)
const [email, setEmail] = useState('');
<input value={email} onChange={e => setEmail(e.target.value)} />

// Uncontrolled (only for trivial cases)
<input ref={inputRef} defaultValue="hello" />
```

**Guideline**: Default to **controlled**. Reach for uncontrolled only when:

* Performance is proven to be an issue, **and**
* Validation/sync needs are minimal.

---

## 7) Validation with Zod (Requests & Responses)

* Define **single-source schemas** with Zod for:

  * **Form inputs** (client)
  * **API route inputs** (server)
  * **API responses** (server → client)

```ts
// schemas/user.ts
import { z } from "zod";

export const UserCreate = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

export const User = UserCreate.extend({
  id: z.string().uuid(),
  createdAt: z.string(),
});
```

**Benefits**

* Early, consistent failures (dev + runtime).
* Shared types via `z.infer<typeof Schema>`.
* Safer refactors and fewer contract mismatches.

---

## 8) Server Data & Caching with TanStack Query

* Read this: [https://tkdodo.eu/blog/effective-react-query-keys#use-query-key-factories](https://tkdodo.eu/blog/effective-react-query-keys#use-query-key-factories)
* Tldr; use featured co-located query key factories. If that doesn't make sense, go read the thing. It's two small paragraphs.
* Use **TanStack Query** for client-side fetching, caching, retries, and background revalidation.
* Keep **server state** in the cache, **UI state** in React.
* Derive query keys from parameters; invalidate precisely.

```tsx
const { data } = useQuery({
  queryKey: [projectsKeys.all()], // projectsKeys.all() would probably look like ["projects"]
  queryFn: () => fetch("/api/projects").then(r => r.json()),
  staleTime: 60_000,
});
```

**Guideline**

* Prefer SSR for initial data; hydrate with **TanStack Query** for live updates.
* Avoid double-fetching; pass SSR data as `initialData`.

---

## 9) Performance: JS vs CSS vs Dev Time

* **Minimize JS shipped to the client**:

  * Favor **Server Components**; mark interactive parts with `"use client"`.
  * Code-split via lazy/dynamic imports for heavy components.
* **CSS**:

  * Prefer utility-first (Tailwind) or framework-level CSS to avoid runtime style overhead.
  * Co-locate styles with components; avoid global leakage.
* **Trade-offs**:

  * If a library saves major dev time with small bundle impact, use it.
  * If it adds >20–30kb gzipped for minor benefit, reconsider. That said, this is always context-dependent. Pre-optimization is never good practice.

---

## 10) Image, Icon, and Font Licensing

* Always verify that assets are **licensed for commercial use**.
* Prefer **CC0 (Public Domain)** or **MIT-like** licensed assets for simplicity.
* For stock images, use trusted CC0 sources like:

  * [Unsplash](https://unsplash.com/license)
  * [Pexels](https://www.pexels.com/license/)
  * [Pixabay](https://pixabay.com/service/license/)
* For **icons**:

  * Use libraries with permissive licenses (e.g., [Lucide](https://lucide.dev/license) under ISC, [Heroicons](https://github.com/tailwindlabs/heroicons) under MIT).
  * Avoid libraries with “attribution required” unless credit is displayed.
* For **fonts**:

  * Verify license (Google Fonts are generally OFL).
  * Never self-host unlicensed fonts or copyrighted branding assets.

**Rule:** If it’s not CC0, MIT, ISC, or OFL — **verify explicitly** before use in any production or commercial project.

---

## 11) Data Flow & Side Effects

* Keep side effects in **Server Actions** or **mutations**; keep components pure.
* Guard every network boundary with **Zod**. (Again, this is very opinianted)
* Centralize **error handling**: user-friendly toasts, logs to server. Generally you shouldn't be doing much with the Next.js server though, the framework is really just for the SSR.

---

## 12) Project Conventions (Quick Hits)

* **TypeScript** everywhere; `strict` mode on.
* **File naming**: `kebab-case` files, `PascalCase` components. (Again, opinionated. Whatever you pick is fine as long as you stick with it)
* **Feature vs layer file structure**: Up to you. Layer is usually easier for smaller projects. Feature is usually better for larger projects.
* **Hooks** start with `use`; one responsibility per hook.
* **Tests** colocated: `Component.test.tsx`, `hook.test.ts`.
* **Accessibility** checks (lint rules, quick keyboard pass).
* **No magic numbers/strings**: extract constants.
* **Comments** explain *why*, not *what*.
