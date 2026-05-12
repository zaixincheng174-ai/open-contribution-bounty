# Review Checklist

Use this checklist to review the runtime upgrades article before publishing or
accepting it as a Track 3 technical blog submission.

## Source accuracy

- The article distinguishes node/client code from on-chain Wasm runtime code.
- The article describes runtime upgrades as forkless on-chain runtime code
  replacement, not as a normal native binary deployment.
- The article treats sudo as a tutorial or local-chain path, not as the default
  production governance model.
- The article avoids claiming that every parachain uses the same exact upgrade
  extrinsic sequence.
- The references point to current Polkadot Developer Docs, Polkadot SDK docs,
  and try-runtime documentation.

## Technical completeness

- `spec_version` bumping is explained as a required release step.
- Runtime API, metadata, events, errors, weights, and downstream tooling are
  included in the upgrade surface.
- Storage migration triggers and risks are described with practical examples.
- `OnRuntimeUpgrade`, `UncheckedOnRuntimeUpgrade`, `VersionedMigration`,
  `pre_upgrade`, `post_upgrade`, and `try-runtime` are covered at a high level.
- The article explains why large or unbounded migrations may need a multi-block
  migration strategy.

## Builder usefulness

- The preparation workflow is ordered enough for a builder to follow.
- The operational checklist includes build provenance, benchmark updates,
  client compatibility, monitoring, and incident response.
- The article gives concrete failure modes reviewers can check before an
  upgrade is submitted.
- The testing guidance emphasizes representative state and migration-specific
  validation instead of relying only on successful compilation.

## Editorial quality

- Headings are clear and scannable.
- Markdown formatting is consistent.
- The article does not include investment, token-price, or promotional claims.
- The tone is technical, neutral, and suitable for an OpenGuild bounty
  submission.
