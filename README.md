# compat-board

Which Bynk releases each repository enrolled in the
[org canary](https://github.com/bynk-lang/.github/blob/main/canary/README.md)
passes with. A Bynk app, deployed to Cloudflare Workers.

The canary runs every enrolled repository's checks against each new Bynk
release, and each run POSTs its result here, authenticated by the GitHub
Actions OIDC token it mints for this board. The board shows the latest result
for each repository and Bynk version.

## Endpoints

| Route | Who | What |
| --- | --- | --- |
| `POST /v2/runs` | a canary run, with its GitHub OIDC token | Record one result. The body and the token are specified in [`canary/REPORTING.md`](https://github.com/bynk-lang/.github/blob/main/canary/REPORTING.md). |
| `GET /board.json` | anyone | `{"runs": [...]}`: every result held, one per repository and version. |
| `GET /` | anyone | The board as a page: repositories down the side, the eight newest Bynk versions across the top, each cell linked to its run. |

`POST /v2/runs`:
- **New result:** `201 {"stored": true}`.
- **No change:** `200 {"stored": false}`. That's for a retried report (the same `run_url`), or one no newer than the result already held for that repository and version. Results are compared by `finished_at`, so of two runs that finish in the same second, the first one reported is kept.
- **No valid token** for this board's audience, or a token whose `sub` isn't a branch run of a bynk-lang repository (another org's, a pull request's): `401`. The token is verified against GitHub's public keys; there's no secret.
- **A report for another repository** than the run that sent it: `403`.
- **Body outside the contract** (an unknown `kind`, a `result` other than `pass` or `fail`, a malformed version, commit, run URL or timestamp): `400`.

All of the board lives in one agent (a Durable Object), so each report is compared with the result it would replace and stored in one step.

## Layout

```text
src/compat/model.bynk    commons compat.model: the report types (REPORTING.md's body), version ordering
src/compat/page.bynk     commons compat.page: the HTML page
src/compat/board.bynk    context compat.board: the CanaryRun actor (OIDC), the Board agent, the HTTP service
tests/compat/model.bynk  version ordering
tests/compat/page.bynk   escaping, columns, the rendered page
tests/compat/board.bynk  storage rules (new, retried, newer, older, per version), OIDC subjects, the 403, the public routes
```

## Run it locally

```sh
bynkc test .
bynk dev
```

Then open <http://localhost:8787/>. Reporting needs a real GitHub Actions OIDC
token minted for this board's audience, so locally the tests are the way to
exercise `POST /v2/runs`. They supply the run's identity with
`by CanaryRun("repo:bynk-lang/…:ref:refs/heads/main")`.

## Deploying

[`deploy.yml`](.github/workflows/deploy.yml) does an offline dry run on every pull request, and deploys on every push to `main`. The deploy needs two repository secrets; until they exist it warns and skips:

| Secret | Value |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | A Cloudflare API token with Workers deploy scope. |
| `CLOUDFLARE_ACCOUNT_ID` | The Cloudflare account ID. |

The Worker is called `compat-board`, so it is served at
`https://compat-board.<your-subdomain>.workers.dev`.

**After the first deploy,** `bynk.deploy.lock` changes, because it records the deployed context. The action never commits, so the workflow uploads the new lock as an artifact named `bynk.deploy.lock`. Commit it.

## Reporting needs no setup

Every enrolled repository's canary workflow grants `id-token: write`, and
`canary.yml` mints the run's OIDC token for this board's URL and POSTs to
`/v2/runs`. There's no shared key or organisation secret, so private
repositories report too.

## The canary

This repository is enrolled itself, as `kind: example`: [`bynk-canary.yml`](.github/workflows/bynk-canary.yml) runs bynk-ci and a bynk-deploy dry run at each new Bynk release.

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE) at
your option.
