# compat-board

Which Bynk releases each repository enrolled in the
[org canary](https://github.com/bynk-lang/.github/blob/main/canary/README.md)
passes with. A Bynk app, deployed to Cloudflare Workers.

The canary runs every enrolled repository's checks against each new Bynk
release. When a repository's canary run has a report URL and key, it POSTs its
result here, and the board shows the latest result for each repository and
Bynk version.

## Endpoints

| Route | Who | What |
| --- | --- | --- |
| `POST /runs` | the canary, signed | Record one result. The body and signature are specified in [`canary/REPORTING.md`](https://github.com/bynk-lang/.github/blob/main/canary/REPORTING.md). |
| `GET /board.json` | anyone | `{"runs": [...]}`: every result held, one per repository and version. |
| `GET /` | anyone | The board as a page: repositories down the side, the eight newest Bynk versions across the top, each cell linked to its run. |

`POST /runs`:
- **New result:** `201 {"stored": true}`.
- **No change:** `200 {"stored": false}`. That's for a retried report (the same `run_url`), or one no newer than the result already held for that repository and version. Results are compared by `finished_at`, so of two runs that finish in the same second, the first one reported is kept.
- **Bad signature,** or a timestamp more than 300 seconds off: `401`.
- **Body outside the contract** (an unknown `kind`, a `result` other than `pass` or `fail`, a malformed version, commit, run URL or timestamp): `400`.

All of the board lives in one agent (a Durable Object), so each report is compared with the result it would replace and stored in one step.

## Layout

```text
src/compat/model.bynk    commons compat.model: the report types (REPORTING.md's body), version ordering
src/compat/page.bynk     commons compat.page: the HTML page
src/compat/board.bynk    context compat.board: the Canary actor, the Board agent, the HTTP service
tests/compat/model.bynk  version ordering
tests/compat/page.bynk   escaping, columns, the rendered page
tests/compat/board.bynk  storage rules (new, retried, newer, older, per version), the public routes
```

## Run it locally

```sh
bynkc test .
bynk dev -- --var COMPAT_BOARD_KEY:dev-key
```

Then post a signed report with the canary's own reporter, from a checkout of
bynk-lang/.github:

```sh
REPORT_URL=http://localhost:8787/runs REPORT_KEY=dev-key \
REPO=bynk-lang/bynk-deploy KIND=action BYNK_VERSION=0.309.4 RESULT=pass \
COMMIT_SHA=0123456789abcdef0123456789abcdef01234567 \
RUN_URL=https://github.com/bynk-lang/bynk-deploy/actions/runs/1 \
  canary/scripts/report.sh
```

and open <http://localhost:8787/>.

## Deploying

[`deploy.yml`](.github/workflows/deploy.yml) does an offline dry run on every pull request, and deploys on every push to `main`. The deploy needs three repository secrets; until they exist it warns and skips:

| Secret | Value |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | A Cloudflare API token with Workers deploy scope. |
| `CLOUDFLARE_ACCOUNT_ID` | The Cloudflare account ID. |
| `COMPAT_BOARD_KEY` | The shared HMAC key, a long random string, e.g. `openssl rand -hex 32`. It's set on the Worker as the `Signature` actor's secret. |

The Worker is called `compat-board`, so it is served at
`https://compat-board.<your-subdomain>.workers.dev`.

**After the first deploy,** `bynk.deploy.lock` changes, because it records the deployed context. The action never commits, so the workflow uploads the new lock as an artifact named `bynk.deploy.lock`. Commit it.

## Turning on reporting

Once the board is deployed, set two **organisation** secrets on `bynk-lang`, available to the enrolled repositories:

| Secret | Value |
| --- | --- |
| `BYNK_CANARY_REPORT_URL` | `https://compat-board.<your-subdomain>.workers.dev/runs` |
| `BYNK_CANARY_REPORT_KEY` | the same value as `COMPAT_BOARD_KEY` |

Every enrolled repository's canary workflow already passes these to `canary.yml`. From the next canary run, each run reports here. Until then, reporting is skipped silently.

## The canary

This repository is enrolled itself, as `kind: example`: [`bynk-canary.yml`](.github/workflows/bynk-canary.yml) runs bynk-ci and a bynk-deploy dry run at each new Bynk release.

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE) at
your option.
