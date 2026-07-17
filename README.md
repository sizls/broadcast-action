# `sizls/broadcast-action`

> Auto-post release announcements to social + community channels from your GitHub Actions release workflow. Every post gets a Pluck-signed receipt so visitors at [`directive.run/broadcast`](https://directive.run/broadcast) can verify the post happened, exactly as shown, months later.
>
> **Roadmap**: Rekor anchoring of the cassette envelope is planned (the `PostsIndexRow.rekorUrl` field ships as `null` today; wiring the anchor lands the third-party-witness leg). The signed cassette + client-side ed25519 verify path is already live.

This is the **mirror** of `@sizl/broadcast-action` — the source lives in the private [`sizls/broadcast`](https://github.com/sizls/broadcast) monorepo and is published here on every release as a bundled `dist/index.js` you can pin by SHA. Every release is signed via [Sigstore](https://sigstore.dev) using GitHub Actions keyless OIDC, so you can verify the binary you're running is the binary the monorepo produced.

## Install (30 seconds)

Add to your release workflow. The SaaS path at `broadcast.run` is the
shortest install — one API key, three platform tokens, done:

```yaml
# .github/workflows/release.yml
on:
  release:
    types: [published]

jobs:
  broadcast:
    runs-on: ubuntu-latest
    steps:
      - uses: sizls/broadcast-action@<sha>   # pin by SHA, not tag
        with:
          platforms: bluesky,mastodon,discord
        env:
          BROADCAST_API_KEY:     ${{ secrets.BROADCAST_API_KEY }}
          BLUESKY_APP_PASSWORD:  ${{ secrets.BLUESKY_APP_PASSWORD }}
          MASTODON_TOKEN:        ${{ secrets.MASTODON_TOKEN }}
          DISCORD_WEBHOOK_URL:   ${{ secrets.DISCORD_WEBHOOK_URL }}
```

Get a `BROADCAST_API_KEY` by emailing `hi@broadcast.run` during the
private beta. Every post comes back with a public receipt URL at
`broadcast.run/r/<token>` that anyone can open to cryptographically
verify the post in their own browser — no signup, no SDK, just
WebCrypto against the inlined cassette envelope.

### Legacy / self-host (no SaaS account)

If you run your own posts-index Worker (deployed via
`pnpm create broadcast-worker` on your own Cloudflare account), skip
`BROADCAST_API_KEY` and use the HMAC inputs instead:

```yaml
      - uses: sizls/broadcast-action@<sha>
        with:
          platforms: bluesky,mastodon,discord
          posts-index-url: https://your-worker.example.com/posts
          posts-index-secret: ${{ secrets.POSTS_INDEX_SECRET }}
        env:
          BLUESKY_APP_PASSWORD: ${{ secrets.BLUESKY_APP_PASSWORD }}
          MASTODON_TOKEN:       ${{ secrets.MASTODON_TOKEN }}
          DISCORD_WEBHOOK_URL:  ${{ secrets.DISCORD_WEBHOOK_URL }}
```

When `BROADCAST_API_KEY` is unset the action falls back to this path
verbatim — pre-existing consumers don't need to migrate.

## What it does

When a GitHub release is published in your project, this Action:

1. Translates the release payload into the broadcast runtime's `AnnounceableEvent` shape.
2. **Resolves the tier** for the event from your `.sizl/broadcast.config.json` (PATCH releases default to Tier 1, MINOR / MAJOR / RECAP / MANUAL default to Tier 2).
3. **Refuses Tier 2/3 events with a structured error** — the GitHub Actions runtime is ephemeral and cannot host the 15-minute Tier 2 approval window. The refusal message points you at the Cloudflare Worker template (`pnpm create broadcast-worker`) which CAN.
4. For Tier 1 events, runs the broadcaster's safety floor (cost circuit breaker, internal-token sanitizer, kill switch, dry-run mode) and dispatches the post.
5. After each successful post, POSTs the cassette envelope (Pluck-signed under the tenant's managed ed25519 keypair; the `rekorUrl` field is placeholder until Sigstore Rekor anchoring wires up) to the Sizl-hosted posts-index at `broadcast.sizls.com/posts`. That endpoint feeds the `broadcast.sizls.com/broadcast` kill-log page and the long-running engagement collector.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `platforms` | yes | — | Comma-separated platform IDs (`bluesky,mastodon,twitter,linkedin,discord,slack,facebook,instagram,threads,reddit,mailchimp,convertkit,beehiiv,buttondown,ghost,mailerlite,substack`). |
| `config-path` | no | `.sizl/broadcast.config.json` | Path to your broadcast config JSON file inside the repo. The Action loads this via `JSON.parse`; `.ts` configs are not supported. |
| `dry-run` | no | `false` | When `true`, validates + sanitizes + signs but doesn't post. |
| `posts-index-url` | no | `https://broadcast.sizls.com/posts` | Sizl-hosted posts-index URL to POST cassette envelopes to. |
| `posts-index-secret` | no | — | HMAC secret for the posts-index POST. Skipped on dry-run. |

## Outputs

| Output | Description |
|---|---|
| `cassette-hashes` | JSON array of `{platform, cassetteHash, postId}` entries — one per posted platform. |
| `skipped` | JSON array of `{platform, reason}` entries for platforms that were skipped (kill-switch, sanitize, validation, idempotency). |
| `receipt-urls` | JSON array of `{platform, receiptUrl}` entries returned by the SaaS path (`broadcast.run/r/<token>` — one per accepted entry). Empty on the legacy HMAC path; receipts there resolve through the kill-log page at the posts-index host. |

## Retry policy — SaaS rejection reasons

The SaaS Worker returns each entry as either `accepted` or `rejected`. Consumers writing custom retry logic can `import { REJECT_REASONS, type RejectReason } from "@sizl/broadcast-posts-index"` and pattern-match against the stable enum. The four failure classes:

| Class | Codes | Retry class |
|---|---|---|
| Shape validation | `entry-not-object`, `invalid-platform`, `invalid-postId`, `invalid-postUrl`, `missing-text`, `text-exceeds-server-cap`, `invalid-text`, `invalid-eventId`, `invalid-briefHash`, `missing-approval`, `invalid-approval-tier`, `invalid-approvedBy`, `invalid-approvedAt`, `invalid-draftsConsidered` | **Terminal** — caller-side payload bug; fix and retry. |
| Batch bounds | `too-many-entries`, `quota-exhausted-in-batch` | **Terminal** — split the batch or wait for the next quota window. |
| Signer-side | `signer-unavailable`, `sign-failed` | **Retryable** — server transient; exponential backoff. |
| Signer contract | `sign-no-cassette` | **Investigate** — signer succeeded without a cassette; report don't retry. |
| Persist | `persist-failed` | **Retryable** — KV write transient; exponential backoff. |

The `invalid-text` reason additionally carries a `detail` string with the byte offset + Unicode code point of the first rejected control character — safe to surface in operator-facing logs.

### Top-level errors (`response.error`)

Reject reasons above live inside `response.rejected[].reason` — one per entry that failed. A separate class of error terminates the whole request before any entry is processed and appears as `response.error` at HTTP 4xx/5xx:

| HTTP | `error` | Retry class |
|---|---|---|
| 400 | `malformed-json` | **Terminal** — caller bug; fix the JSON. |
| 400 | `no-entries` | **Terminal** — send at least one entry. |
| 400 | `too-many-entries` | **Terminal** — split the batch (cap is 32). |
| 413 | `body-too-large` | **Terminal** — split the batch (cap is 200 KiB). |
| 429 | `monthly-cap-reached` | **Wait** — monthly usage counter hit the tier cap. |
| 429 | `rate-limit-exceeded` | **Retryable with backoff** — burst limit (60/min); back off and retry. |
| 500 | `signer-misconfigured` | **Investigate** — server-side configuration issue; report don't retry. |

## Verify the binary

Every release of this Action is signed with Sigstore keyless OIDC tied to the source workflow in `sizls/broadcast`. Before pinning a new SHA, run:

```bash
cosign verify-blob \
  --certificate-identity-regexp "https://github.com/sizls/broadcast/\\.github/workflows/release-action\\.yml@.+" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  --signature dist/index.js.sig \
  dist/index.js
```

The certificate-identity regex anchors the signature to the source workflow path — even if an attacker compromises the public mirror repo and tries to push an unsigned binary, `cosign verify-blob` will fail.

**Always pin by SHA, not tag.** Tags can be retargeted. SHAs cannot. The pinned-SHA pattern matches industry norms for production GitHub Actions (see `dependabot` and `renovate` defaults).

```yaml
# good
- uses: sizls/broadcast-action@a1b2c3d4...
# avoid
- uses: sizls/broadcast-action@v1
- uses: sizls/broadcast-action@main
```

## Why a separate public repo?

The source — adapter network, cost-breaker logic, sanitizer rules — lives in the private `sizls/broadcast` monorepo because the moat is the Sizl-curated adapter + Bureau-signing surface, not any single bundle. This public mirror exists because GitHub Actions resolves `uses: <repo>@<sha>` against a public repository path. Mirroring the bundled artifact here (and ONLY the artifact, no source) is the canonical pattern that keeps the source posture private while letting the Action be consumed externally.

## Docs

- [`directive.run/docs/broadcast/action`](https://directive.run/docs/broadcast/action) — full reference
- [`directive.run/docs/broadcast/worker`](https://directive.run/docs/broadcast/worker) — Cloudflare Worker template (Tier 2/3 events)
- [`directive.run/docs/broadcast/cli`](https://directive.run/docs/broadcast/cli) — `@sizl/broadcast` CLI for manual posts + backfills
- [`directive.run/broadcast`](https://directive.run/broadcast) — public Pluck-verifiable kill log

## License

Apache-2.0 — see [LICENSE.md](./LICENSE.md).

The source remains under a separate license inside the private monorepo. This mirror's license applies only to the published bundle.

## Security

Found a security issue? Don't open a public issue — email `security@directive.run` or use [GitHub's private vulnerability reporting](https://github.com/sizls/broadcast-action/security/advisories/new).

---

*Built by [Sizl](https://sizl.dev) on [Directive](https://directive.run) and [Pluck](https://pluck.run).*
