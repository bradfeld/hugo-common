# hugo-common

Shared infrastructure for the Feld Hugo-on-Vercel content network
(feld.com, foundry.vc, adventuresinclaude.ai, zeroknowledge.ink, …).

This repo is the **single source of truth** for how every site pins Hugo, builds on
Vercel, auto-updates, and verifies itself — so the version can't drift or be forgotten
(the failure that built zeroknowledge.ink on Hugo 0.58.2 and emptied its byline), and
new sites start consistent.

Each site consumes it two ways: (1) its **Renovate preset** and **reusable smoke
workflow** by GitHub reference — `extends: github>bradfeld/hugo-common` and
`uses: bradfeld/hugo-common/.github/workflows/smoke.yml@main` (active on all four sites
today); and (2) for the **shared partials**, as a **git submodule** at `themes/hugo-common`
listed in the site's `theme` config. (Git submodule, not Hugo Module — Go isn't installed;
the submodule is the proven PaperMod pattern on this Vercel setup.) The submodule/partial
wiring is the next rollout step — see status below.

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

## Gotchas (learned the hard way)

- **pnpm 11 silently skips the build script.** `corepack` can pull pnpm 11, which
  IGNORES `package.json`'s `pnpm.onlyBuiltDependencies` (`ERR_PNPM_IGNORED_BUILDS`) —
  hugo-extended's postinstall never runs, there's no `node_modules/.bin/hugo`, and the
  build dies. The reusable smoke workflow pins **pnpm@10** to dodge this. For forward-
  compat on a real deploy, move `onlyBuiltDependencies` to `pnpm-workspace.yaml` or pin
  pnpm via `packageManager` in `package.json`.
- **A green build is not a correct render.** The byline bug that started all this BUILT
  clean on the wrong Hugo version. That's why the smoke gate asserts render invariants
  (non-empty critical elements), not just exit 0.
- **PaperMod's accessible card links are intentionally empty `<a>`** — they carry an
  `aria-label` instead of text. The smoke gate only fails on anchors missing BOTH text
  and aria-label (the genuine author-bug class), not these.
- **Don't switch a site's package manager during migration.** zeroknowledge's serverless
  `/api` functions broke when pnpm→npm changed the `node_modules` layout Vercel's
  `@vercel/node` builder expected (`@noble` ENOENT). Sites with functions keep their
  existing manager; pure-static sites can standardize on npm.
- **Verifying a Vercel deploy via the API:** commit messages can carry unescaped control
  chars that break `jq` — pipe through `perl -pe 's/[\x00-\x1f]//g'` first (BSD/macOS
  `tr -d '\000-\037'` and `tr -d '[:cntrl:]'` do NOT reliably strip them). Token:
  `gcloud secrets versions access latest --secret=platform_vercel_token --project=authormagic-480416`;
  team id lives in `platform_vercel_team_id`.

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

---

## Rollout status (2026-06-13)

All four sites migrated to the in-repo Hugo pin (`0.163.0`), the shared Renovate preset,
and the reusable smoke gate — each deployed and verified live.

| site | pin | pkg mgr | renovate + smoke | live |
|---|---|---|---|---|
| zeroknowledge.ink | 0.163.0 | pnpm (has `/api`) | ✓ | ✓ |
| foundry.vc | 0.163.0 | npm | ✓ | ✓ |
| adventuresinclaude.ai | 0.163.0 | npm | ✓ | ✓ |
| feld.com | 0.163.0 | npm | ✓ | ✓ |

**Still open:**

- **Renovate GitHub App** isn't installed on the bradfeld repos yet (Brad's one-time
  click). Until then, auto-bump PRs don't open. Dependabot is the zero-install fallback.
- **Vercel `HUGO_VERSION` env cleanup** — the old hidden per-project env pins on
  feld/foundry/aic are now redundant (the in-repo pin is authoritative). Remove them, and
  fix foundry's misconfigured `sensitive`-prod one. Expand-contract: the in-repo pin is
  proven first (it is).
- **zk pnpm-11 forward-compat** — move `onlyBuiltDependencies` to `pnpm-workspace.yaml`
  or pin `packageManager` (see Gotchas).
- **`bradfeld/hugo-site-template`** scaffold repo — not yet created.
- **Submodule / partial wiring** — the shared `preview-noindex` partial isn't vendored
  into any site yet (sites still carry inline noindex logic). Thin win, deferred.
