# Deploying Fink

## General Translation production

Deploy only to account `00246eb74c164b5dbee7d3ee028463c7`, explicitly set as `CLOUDFLARE_ACCOUNT_ID`. The deploy script rejects a different account ID. Use Node.js 22+ and pnpm 10.23.0 with the frozen lockfile. Wrangler is pinned to 4.148.0.

```sh
pnpm install --frozen-lockfile
pnpm build
CLOUDFLARE_ACCOUNT_ID=00246eb74c164b5dbee7d3ee028463c7 pnpm exec wrangler deploy --config wrangler.jsonc --name fink --domain fink.inlang.com --secrets-file /private/path/fink-worker-secrets.json
```

The private JSON file must contain `GITHUB_CLIENT_SECRET` and a newly generated `SESSION_SECRET` of at least 32 characters. Keep values in 1Password and inject them securely; never commit or print them. Wrangler OAuth or a destination-scoped Cloudflare API token authorizes deployment and is not a Worker binding. `scripts/deploy.mjs` accepts the two secrets through the environment, writes a mode-0600 temporary secrets file, strips them from the child environment, and deletes the file after deployment.

Production uses the transferred, General Translation-owned `inlang` GitHub App, ID `409404`, client ID `Iv1.436da58906364f96`. The production callback URL registered with that App must be:

`https://fink.inlang.com/api/auth/callback`

`GITHUB_CALLBACK_ORIGIN=https://fink.inlang.com` uses the production Worker as the stable callback for buyer-owned workers.dev previews too. `WORKERS_SUBDOMAIN=generaltranslation` limits the accepted preview origins to this destination account. No Opral callback Worker is required. Production and any optional previews must share the new session secret to relay login between origins.

The Worker serves `worker/index.ts` and the `dist` asset directory bound as `ASSETS`. Preserve SPA fallback and run-worker-first paths `/api/*`, `/ingest/*`, `/docs/*`, `/compare/*`. Compatibility date is `2026-10-07` with `nodejs_compat`. No KV, R2, D1, Durable Object, queue, or service bindings are required.

The SDK's large Lix engine is compressed by the Vite adapter into a roughly 19 MiB gzip asset and decompressed in the browser before WASM compilation. The build validates the upstream loader shape and `prepare-assets.mjs` excludes oversized originals from upload. SDK and Lix versions remain pinned.

`pnpm deploy:production` builds and deploys `fink` with its production Custom Domain. Optional preview targets do not attach that domain.

## GitHub Actions environments

The workflow uses the repository's `production` and `preview` deployment environments. Store these environment secrets in each:

- `CLOUDFLARE_API_TOKEN`: a General Translation-owned API token scoped to destination account `00246eb74c164b5dbee7d3ee028463c7`. Production needs access to deploy `fink` and manage its Custom Domain in the `inlang.com` zone. Preview deployment and cleanup need access to create, update, and delete `fink-pr-*` Workers in the destination account.
- `FINK_GITHUB_CLIENT_SECRET`: the transferred `inlang` App's client secret, backed up in 1Password's `Sensitive Secrets / inlang GitHub App` item. The workflow exposes it as `GITHUB_CLIENT_SECRET` only in the deployment step; GitHub reserves secret names beginning with `GITHUB_`.
- `FINK_SESSION_SECRET`: the existing destination production session-signing secret from the concealed `Fink SESSION_SECRET` field in that item. The workflow maps it to `SESSION_SECRET`. Production and previews must use the same value for the stable production callback flow.

The account ID, App client ID, Workers subdomain, and callback origin are non-secret values pinned in the deployment workflow/configuration. No Infisical account, identity, repository variables, or OIDC grant is needed.

Production deployment runs from `main`; keep the `production` environment's existing `main` branch policy. Same-repository pull requests deploy into `preview`; forks cannot deploy or run cleanup. Secret values are passed only to deployment/cleanup steps, after installation and tests. The cleanup job receives only the Cloudflare token. Both jobs use pnpm's frozen lockfile and the repository-pinned Wrangler.

Changing environment secrets affects future deployments. Keep both environments synchronized when rotating the App client secret or session secret.

Validate HTTPS, editor initialization, compressed WASM loading, GitHub sign-in, and read/update access to an explicitly authorized test repository before considering authentication complete.

## Extracted history

The extraction starts at `opral/inlang` commit `494db8fe8f4923ce5c5f68f6f0934d9054e288f3`. `git-filter-repo` retained 240 commits affecting historical Fink paths and moved their contents to the repository root. The destination repository's history was merged rather than replaced.

```sh
git filter-repo --force \
  --path packages/fink/ --path packages/fink2/ \
  --path inlang/packages/fink/ --path inlang/source-code/fink2/ \
  --path-rename packages/fink/: --path-rename packages/fink2/: \
  --path-rename inlang/packages/fink/: --path-rename inlang/source-code/fink2/: \
  --refs main
```

Hashes change during filtering. The extraction merge precedes the standalone implementation so the original prototype history remains inspectable.
