# Cloudflare Self-Hosting: Workers Builds

Fork-local doc. Upstream (`every-app/open-seo`) does not have this file — see
[Surviving an upstream update](#surviving-an-upstream-update) before you follow
the update flow in [Operations](./SELF_HOSTING_CLOUDFLARE_OPERATIONS.md).

Workers Builds deploys this repo to the Worker on every push to `main`, instead
of running `pnpm run deploy` from a laptop. The dashboard is the source of
truth for these settings; this file records what they are and why.

## Configuration

`Compute` -> `Workers & Pages` -> your Worker -> `Settings` -> `Build`.

| Field                                | Value                                                                 |
| ------------------------------------ | --------------------------------------------------------------------- |
| Git account                          | `felipewilliam2`                                                      |
| Repository                           | `open-seo`                                                            |
| Production branch                    | `main`                                                                |
| Builds for non-production branches   | on                                                                    |
| Build command                        | `pnpm run build`                                                      |
| Deploy command                       | `npx wrangler d1 migrations apply DB --remote && npx wrangler deploy` |
| Non-production branch deploy command | `npx wrangler versions upload`                                        |
| Root directory                       | repo root (default)                                                   |

Nothing goes in `Build variables`. The heap bump the SSR build needs already
comes from `.npmrc` (`node-options=--max-old-space-size=4096`), and `AUTH_MODE`
left unset resolves to `cloudflare_access` — see `src/lib/auth-mode.ts`, where
the schema is `.catch("cloudflare_access")`. An unset auth mode fails closed.

## Why the deploy command runs migrations

`package.json`'s `deploy` script is three steps:

```
db:migrate:prod && build && wrangler deploy
```

Workers Builds splits build and deploy into separate fields, so a deploy command
of just `npx wrangler deploy` silently drops `db:migrate:prod`. Nothing breaks
the day you set it up — it breaks on the first upstream update that adds a D1
migration, when the Worker starts serving code that expects tables the database
does not have.

Applying migrations from the deploy command requires the build's API token to
have D1 edit permission (`Advanced settings` -> `API Token`).

The migration step is idempotent: `wrangler d1 migrations apply` compares
`drizzle/` against the `d1_migrations` table and is a no-op when they match.

## Why non-production branches upload instead of deploy

`wrangler versions upload` uploads a version without promoting it to production.
If that field inherits `npx wrangler deploy` instead, every push to any branch —
`claude/*`, `chore/*`, a release branch — publishes straight to production.

## What the deploy reads

`npx wrangler deploy` reads `wrangler.jsonc` from the repo, so the bindings it
targets are whatever that file says. This fork points them at this account's
resources (the `-anhanga` suffix); upstream's copy points at resources in an
account we do not own. Keep that edit — deploying with upstream's ids either
fails or targets the wrong place.

Secrets are not in the repo and `wrangler deploy` preserves them. The six on the
Worker (`BETTER_AUTH_SECRET`, `DATAFORSEO_API_KEY`, `GOOGLE_CLIENT_ID`,
`GOOGLE_CLIENT_SECRET`, `POLICY_AUD`, `TEAM_DOMAIN`) survive every deploy.

## Where the app is served

Two hostnames reach the same Worker:

- `https://seo.anhanga.tech` — Custom Domain, the one to use
- `https://open-seo.withered-shape-65d6.workers.dev` — the workers.dev route
  `wrangler deploy` prints

Neither appears in `wrangler.jsonc`: there is no `routes` block and no
`workers_dev` key. Custom Domains live in the Worker's configuration
(`Settings` -> `Domains & Routes`), not in the repo, which is why a
`wrangler deploy` with no `routes` block does not remove them. Do not add a
`routes` block to "fix" the apparent omission — an incomplete one would drop the
domain that is currently working.

Both hostnames sit behind the same Cloudflare Access application
(`anhanga.cloudflareaccess.com`). An unauthenticated request to either returns a
302 to the Access login, so neither is a way around the other.

After a deploy, confirming the Custom Domain still answers is one command:

```bash
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://seo.anhanga.tech/
```

A `302` pointing at `anhanga.cloudflareaccess.com` is the healthy answer — the
Worker is up and Access is enforcing. A `200` would mean Access is not in front
of it.

## Surviving an upstream update

The update flow in [Operations](./SELF_HOSTING_CLOUDFLARE_OPERATIONS.md) runs
`git reset --hard upstream/main`, which deletes every file upstream does not
have — including this one — and then force-pushes. Two consequences:

1. Back up this file alongside `wrangler.jsonc`, or it disappears on the next
   update.
2. With Git connected, that force-push **triggers a production deploy**. It is
   no longer a local-only step.

This clone has no `upstream` remote yet (`origin` is the fork). Add it once:

```bash
git remote add upstream https://github.com/every-app/open-seo.git
```

Adjusted flow:

```bash
git fetch upstream
cp wrangler.jsonc /tmp/wrangler.local.jsonc
cp docs/SELF_HOSTING_CLOUDFLARE_WORKERS_BUILDS.md /tmp/workers-builds.md
git checkout main
git reset --hard upstream/main
cp /tmp/wrangler.local.jsonc wrangler.jsonc
cp /tmp/workers-builds.md docs/SELF_HOSTING_CLOUDFLARE_WORKERS_BUILDS.md
git add wrangler.jsonc docs/SELF_HOSTING_CLOUDFLARE_WORKERS_BUILDS.md
git commit -m "restore Cloudflare settings"
git push --force-with-lease origin main
```

Run it on `main`. `git reset --hard` on the wrong branch discards that branch's
work.

## Deploying without Workers Builds

The local path still works and is the one
[Manual deploy](./SELF_HOSTING_CLOUDFLARE_MANUAL.md) documents:

```bash
pnpm install
pnpm run deploy
```

That runs migrations, build, and deploy in one command.
