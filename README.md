# status

Status of the bynk-lang org's live Bynk services, checked every five minutes.
A Bynk app, deployed to Cloudflare Workers.

The canary tests each service's *code* against new Bynk releases. This checks
that the *running* services answer, between canary runs.

## What it checks

| Target | URL | Up when |
| --- | --- | --- |
| compat-board | `https://compat-board.bynk-lang.dev/board.json` | `200` |
| examples-index | `https://examples-index.bynk-lang.dev/examples.json` | `200` |
| registry | `https://raw.githubusercontent.com/bynk-lang/.github/main/canary/repos.json` | `200` (the examples index depends on it) |

The list is `targets()` in [`src/status/model.bynk`](src/status/model.bynk);
change it with a pull request. The service doesn't check itself.

## Endpoints

| Route | What |
| --- | --- |
| `GET /` | A card per target, with: up or down; the last status and response time; uptime over 24 hours and 7 days; when it last failed; and a strip of its last 48 checks (four hours). |
| `GET /status.json` | The same data as a list of summaries: `name`, `url`, `checked`, `up`, `status`, `ms`, `last_at`, `uptime_24h`, `uptime_7d`, `last_failure_at` and `recent`. |

In `status.json`:
- **Uptime** is a percentage scaled by ten (`999` is 99.9%), or `-1` with no checks in that window.
- **`status`** is `0` when the request never completed.
- **`last_failure_at`** is `0` when the target hasn't failed in the last week.
- **Times** are epoch milliseconds.

## How it works

```text
src/status/model.bynk    commons status.model: targets, Check, Summary, uptime and summaries
src/status/page.bynk     commons status.page: the HTML page
src/status/monitor.bynk  context status.monitor: the Probe capability, the Monitor agent, the cron and HTTP services
tests/status/*.bynk      one suite per unit
```

- **A cron run every five minutes** (`service checks from cron`) probes every target concurrently (`parTraverse`), and records each result in that target's `Monitor`. A target being down is a result to record, never a failed run.
- **`Probe`** is a one-operation capability provided over `Fetch` and `Clock`. It times the request, and treats a network error or a timeout as down with status `0`. It's the only network seam, so tests stub it.
- **`Monitor(name)`** is one agent instance per target, holding `checks: Log[Check] @retain(7.days)`.

Why one agent per target: an agent stores all of its state as one value, so
each target's history is kept in its own instance. A full week of one
target's checks (2,016 at one every five minutes) encodes to about 108 KB, or
about 150 KB as stored, well within a Durable Object's value limit. A test
measures it.

## Run it locally

```sh
bynkc test .
bynk dev -- --test-scheduled
```

Then fire the cron by hand, and open <http://localhost:8787/>:

```sh
curl "localhost:8787/__scheduled?cron=*/5+*+*+*+*"
```

## Deploying

[`deploy.yml`](.github/workflows/deploy.yml) does an offline dry run on every
pull request, and deploys on every push to `main` once these repository
secrets exist (until then it warns and skips):

| Secret | Value |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | A Cloudflare API token with Workers deploy scope. |
| `CLOUDFLARE_ACCOUNT_ID` | The Cloudflare account ID. |

The Worker is called `status-monitor`. Its cron trigger (`*/5 * * * *`) is
in the generated `wrangler.toml`. After the first deploy, commit the
`bynk.deploy.lock` that the workflow uploads as an artifact.

## The canary

This repository is enrolled as `kind: example`:
[`bynk-canary.yml`](.github/workflows/bynk-canary.yml) runs bynk-ci and a
bynk-deploy dry run at each new Bynk release.

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE) at
your option.
