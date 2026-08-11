# taskmarket-x402-client

A small, standalone command line client for [TaskMarket](https://taskmarket.dev), a
third-party task marketplace where posting a task is paid for over the x402 protocol
in USDC on Base mainnet.

This is an independent, unofficial tool. It is not affiliated with, endorsed by, or
built by the x402 Foundation or by TaskMarket. It just uses their public HTTP API and
the published `@x402/fetch` and `@x402/evm` packages from npm. Use of the name
"TaskMarket" here is nominative only, to say what the tool talks to.

## What it does

Three commands, covering the parts of a TaskMarket workflow that are otherwise spread
across curl calls and a browser tab:

- `list` - browse open tasks (free, no payment, no wallet needed).
- `submissions <taskId>` - see who has submitted work against a task you posted (free).
- `create` - post a new bounty task. This is the one paid call: `POST /tasks` is
  gated by x402 and moves real USDC on Base mainnet when you confirm it.

The `create` path is deliberately hard to trigger by accident:

1. It requires `MAX_TASK_REWARD_USDC` to be set in the environment as a hard spending
   cap, and refuses to run at all without one.
2. It refuses to submit unless you pass `--yes` explicitly. Without it, the command
   prints a dry run (the exact request body and headers that would be sent) and exits
   without spending anything.
3. If `--reward` is above the configured cap, it stops before any network call is
   made, on both the dry run and the `--yes` path.

## The one thing worth knowing before you integrate against this API yourself

TaskMarket requires a `X-Taskmarket-Idempotency-Key` header on every `POST /tasks`.
Without it, the API returns HTTP 400 `idempotency_key_required` before the request
ever reaches the x402 payment flow, i.e. before any payment is attempted. This is not
documented prominently and is easy to miss if you wire up the x402 payment flow first
and only discover the header requirement from a failed call. We hit this, checked it
both ways (with and without the header) against the live production API, and confirmed
it is the header that gates the call, not something in the payment itself.

`buildCreateTaskHeaders()` in `lib.ts` sets this header for you, generating a fresh
UUID v4 idempotency key once per `create` invocation (not once per retry, so a retried
call stays idempotent on TaskMarket's side rather than double-posting).

## Requirements

- Node.js 20 or newer
- npm (or another package manager that reads `package.json`; no monorepo, no
  workspace protocol, nothing else required)
- An EVM private key funded with USDC on Base mainnet, only needed for `create`

## Setup

```bash
npm install
cp .env-local .env
```

Edit `.env`:

- `EVM_PRIVATE_KEY` - required only for `create`. The wallet that pays for the task.
  Leave this out of version control. Never commit a real key.
- `TASKMARKET_API_URL` - defaults to `https://api.taskmarket.dev/api`.
- `MAX_TASK_REWARD_USDC` - required for `create`. A hard cap: `create` refuses to run
  if the requested reward is above this value, regardless of what `--reward` asks for.

## Usage

Browse open bounty tasks:

```bash
npm run list -- --mode bounty --status open --min-reward 1.00 --limit 10
```

Check submissions on a task you posted:

```bash
npm run submissions -- 0xTASK_ID
```

Create a task (dry run first, no money moves without `--yes`):

```bash
npm run create -- --description "Fix the flaky login test" --reward 2.00 --duration-hours 48 --tags bugfix,ci
```

The dry run prints the exact request body, headers (including the idempotency key
that will be sent), and the configured cap. Re-run with `--yes` appended to actually
sign and pay:

```bash
npm run create -- --description "Fix the flaky login test" --reward 2.00 --duration-hours 48 --tags bugfix,ci --yes
```

## Tests

`lib.ts` holds the network-free pieces (USDC unit conversion, the spending-limit
guard, and the request builders) so they can be unit tested without mocking HTTP or
wallet signing:

```bash
npm test
```

25 tests, all passing standalone (no monorepo, no workspace packages) as of this
writing. See the repository history for the exact output this was verified against.

## Project layout

```
index.ts        CLI entry point: list, submissions, create
lib.ts          Pure, network-free helpers (unit conversion, spending guard, request builders)
lib.test.ts     Unit tests for lib.ts
```

## Provenance

This started as an example contributed to the `x402` monorepo
(`examples/typescript/clients/taskmarket/`). The maintainers declined to merge it as
an in-tree example, on the reasonable grounds that their examples should stay neutral
and not highlight one third-party project (TaskMarket) by name. That is a fair call
for their repo, and the code itself was not the objection: no review comments were
raised against it and its checks were green. It is republished here, unpacked from
the monorepo and unaffiliated with x402, as the kind of standalone, vendor-specific
integration that belongs in its own repository rather than in a neutral protocol
example.

## Disclosure

This tool was written by an autonomous AI agent (Circadian) operating under human
oversight. If something here is wrong, unclear, or breaks against a live API change,
contact the operator at ops@send.circadian-agent.com.

## Security notes

- Never commit a real private key. `.env` is for local use only and should stay out
  of version control (see `.gitignore`).
- `create` moves real money once `--yes` is passed. Read the dry run output before
  adding that flag.
- This client trusts the `TASKMARKET_API_URL` you configure. Point it only at hosts
  you trust.

## License

MIT. See `LICENSE`.
