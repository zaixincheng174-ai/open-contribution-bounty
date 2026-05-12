# Review Checklist

Use this checklist to review the off-chain indexing article before publishing
or accepting it as a Track 3 technical blog submission.

## Source accuracy

- The article describes off-chain indexing as runtime-driven writes to a
  node-local Offchain DB.
- The article clearly separates off-chain indexed data from consensus pallet
  storage.
- The article does not claim that off-chain indexing replaces external
  database-backed indexers.
- The article explains that indexed data availability depends on node
  configuration and local persistence.
- References point to current Polkadot SDK and Polkadot Developer Docs pages.

## Technical completeness

- `sp_io::offchain_index` is introduced.
- Key design, namespacing, versioning, encoding, and payload size are covered.
- Fork/reorg/replay risks are discussed.
- On-chain commitments and local verification are included.
- Testing guidance covers missing, stale, duplicate, and mismatched local data.

## Builder usefulness

- The example architecture shows how to keep a commitment on-chain while
  storing a larger local payload off-chain.
- The article gives concrete key-design ingredients.
- The security boundary is explicit enough for a reviewer to audit a pallet
  design.
- The article helps builders decide when not to use off-chain indexing.

## Editorial quality

- Headings are clear and scannable.
- Markdown formatting is consistent.
- The article avoids token-price, investment, or promotional claims.
- The tone is technical, neutral, and suitable for an OpenGuild bounty
  submission.
