---
name: pattern-shared-ui-labels-as-props
description: In a monorepo with a shared UI package consumed by multiple Next.js apps, keep i18n/context-dependent libraries out of the shared package — pass strings as props, isolate risky cross-instance deps in their own file.
metadata:
  type: feedback
---

**Pattern:** a shared UI package (`packages/ui` / `@est8/ui`-style) that multiple apps consume must NEVER import a library that relies on an app-mounted React Context (i18n libraries like `next-intl`, form-context libraries, etc.) directly. The shared package resolves its OWN copy of that library, which creates its OWN Context object at module-load time — the consuming app's provider (mounted with the APP's copy of the library) never populates it, so `useX()` inside the shared component throws/warns "no context found" at runtime even though typecheck stays green.

**Why:** confirmed twice on Est8Core (`TransferWidget` crashed on `next-intl`'s `useTranslations()` inside `@est8/ui`; would have repeated for a File Viewer component). The failure is invisible to `tsc`/build — it only surfaces at runtime, in the browser, in production-like conditions.

**How to apply:**
1. Shared component takes a `labels` prop (typed interface, e.g. `FooLabels`), never imports the i18n library itself.
2. The consuming app writes a thin wrapper (e.g. `FooMount.tsx`) that calls its own `useTranslations()`/equivalent and passes the built `labels` object down. One wrapper per app.
3. This is safe to do for EVERY shared component with translated strings — treat it as the default, not an exception.
4. Corollary — NOT every library needs this treatment: a library that renders standalone with no requirement that the CONSUMING APP mount a provider (e.g. `react-markdown` — it needs no app-level `<Provider>`) is safe to import directly inside the shared package even if the monorepo has two different `react` copies across packages (a bundler/Next.js dedupes the actual React runtime in practice — verified via existing hook-using shared components already working live). The distinguishing question is not "does this library use React internals" but "does it require the HOST APP to mount something for it to work" — next-intl needs `NextIntlClientProvider` from the app; markdown rendering needs nothing from the app.
5. For the one case that's genuinely uncertain (e.g. a large third-party renderer, new to the stack), isolate it in its OWN file inside the shared package (not inlined into the main component) so that IF it ever does misbehave across the package/app boundary, swapping it for an app-supplied render-prop is a one-file change, not a rewrite. Don't export that internal file from the package's public barrel — keep its types out of the public API surface so other apps in the monorepo aren't forced to typecheck against it.
