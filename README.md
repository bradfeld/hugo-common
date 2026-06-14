# hugo-common

Shared infrastructure for the Feld Hugo-on-Vercel content network
(feld.com, foundry.vc, adventuresinclaude.ai, zeroknowledge.ink, …).

This repo is the **single source of truth** for how every site pins Hugo, builds on
Vercel, auto-updates, and verifies itself — so the version can't drift or be forgotten
(the failure that built zeroknowledge.ink on Hugo 0.58.2 and emptied its byline), and
new sites start consistent.

It is vendored into each site as a **git submodule** at `themes/hugo-common` and listed
in the site's `theme` config. (Git submodule, not Hugo Module — Go isn't installed; the
submodule is the proven PaperMod pattern on this Vercel setup.)

---

## The standard (every site follows this)

**1. Pin Hugo in-repo, not via Vercel env vars.** The version lives in `package.json`
as `hugo-extended` — version-controlled, auditable in git, matching local dev, and
**watchable by Renovate** (a bot can watch an npm dep; it can't watch a hidden Vercel
env var). The standard version is **0.163.0**.

```jsonc
// package.json
{
  "devDependencies": { "hugo-extended": "0.163.0" }
  // Sites with serverless /api functions (e.g. zeroknowledge) ALSO need:
  // "pnpm": { "onlyBuiltDependencies": ["hugo-extended"] }
}
```

**2. Build with the pinned binary** — explicit `node_modules/.bin/hugo`, never bare
`hugo` (Vercel still installs its own 0.58.2 default on PATH). Use the env-aware
`-b` so preview deploys get correct URLs:

```jsonc
// vercel.json
{
  "framework": "hugo",
  "installCommand": "npm install",          // pnpm install --prod=false for sites with functions
  "buildCommand": "if [ \"$VERCEL_ENV\" = production ]; then node_modules/.bin/hugo --gc --minify -b \"https://$VERCEL_PROJECT_PRODUCTION_URL/\"; else node_modules/.bin/hugo --gc --minify -b \"https://$VERCEL_URL/\"; fi",
  "outputDirectory": "public"
}
```

**3. Package manager.** Pure-static sites: **npm** (simplest; runs hugo-extended's
postinstall). Sites with serverless `/api` functions: **keep their existing manager**
(switching breaks Vercel's function dependency resolution — see zeroknowledge's `@noble`
ENOENT). pnpm needs `pnpm.onlyBuiltDependencies: ["hugo-extended"]` to run the postinstall.

**4. Auto-update** via Renovate (`renovate.json` extends this repo's preset):

```json
{ "extends": ["github>bradfeld/hugo-common"] }
```

Renovate opens a PR on each new Hugo release. The site's **smoke workflow** (below)
verifies the bump RENDERS correctly — a green *build* is not enough (the byline bug
built clean). Automerge minor/patch on green; majors get a look.
**Renovate's GitHub App must be installed on the bradfeld repos (one-time).**

**5. Smoke check** — each site's `.github/workflows/smoke.yml` calls the reusable
workflow here:

```yaml
jobs:
  smoke:
    uses: bradfeld/hugo-common/.github/workflows/smoke.yml@main
```

It builds with the pinned Hugo and asserts render invariants (pages built, no empty
critical elements, no Hugo errors) — catching the silent-render regression class.

---

## Adding a new site

1. Start from `bradfeld/hugo-site-template` (it ships steps 1–5 above).
2. `git submodule add https://github.com/bradfeld/hugo-common themes/hugo-common`
3. `theme = ["hugo-common"]` (or `["YourTheme", "hugo-common"]`) in `hugo.toml`.
4. Done — pinned, auto-updating, smoke-verified from day one.

## Partials provided

| partial | purpose |
|---|---|
| `preview-noindex.html` | `<meta noindex>` on any non-production baseURL (set `params.productionHost`) |

_(More infra partials — analytics, shared OG/meta — land here as they're factored out of the bespoke sites.)_
