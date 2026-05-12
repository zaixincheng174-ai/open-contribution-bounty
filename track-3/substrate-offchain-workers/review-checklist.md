# Review Checklist

Use this checklist to review the off-chain workers article before publishing or
accepting it as a Track 3 technical blog submission.

## Source accuracy

- The article explains that off-chain workers run outside deterministic block
  execution after block import.
- The article does not claim that off-chain workers directly mutate on-chain
  storage.
- Signed, unsigned, and unsigned-with-validation transaction paths are
  separated clearly.
- The article treats local off-chain storage and off-chain indexing as
  node-local/non-consensus data, not shared chain state.
- References point to current Polkadot SDK documentation.

## Technical completeness

- The `offchain_worker` hook is introduced.
- HTTP requests, local storage, off-chain indexing, transaction submission, and
  runtime validation are covered.
- `ValidateUnsigned` risk is discussed.
- Duplicate work, scheduling, replay protection, stale data, and key management
  are included.
- Testing guidance covers both worker behavior and runtime validation.

## Builder usefulness

- The implementation flow is ordered enough for a builder to follow.
- The operational checklist highlights production failure modes.
- The article explains when signed transactions are safer than unsigned ones.
- The article gives concrete mitigations for spam, duplicate submissions, and
  unreliable external APIs.

## Editorial quality

- Headings are clear and scannable.
- Markdown formatting is consistent.
- The article avoids token-price, investment, or promotional claims.
- The tone is technical, neutral, and suitable for an OpenGuild bounty
  submission.
