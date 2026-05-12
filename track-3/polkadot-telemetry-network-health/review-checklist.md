# Review Checklist

Use this checklist to review the Polkadot telemetry and network-health article
before publishing or accepting it as a Track 3 technical blog submission.

## Source accuracy

- The article describes Telemetry as a visibility layer, not as a complete
  monitoring or alerting system.
- The article correctly separates Telemetry, local logs, Prometheus metrics,
  RPC checks, and chain observation.
- The article explains that the final value in `--telemetry-url` is a verbosity
  level, not part of the WebSocket URL.
- The article avoids fixed peer-count or validator-count claims that can drift.
- The references use current official Polkadot, Substrate Telemetry, and
  Prometheus resources.

## Technical completeness

- The article covers sync status, best block, finalized block, peer count,
  node version, node identity, and RPC health.
- The article includes practical troubleshooting for missing telemetry, low
  peers, stalled finality, and RPC issues.
- The article includes a production monitoring baseline beyond Telemetry.
- The article discusses security and privacy considerations for node names,
  validator topology, Prometheus, and RPC exposure.

## Builder usefulness

- Operators can turn the monitoring baseline into real alerts and runbooks.
- Developers can understand why user-facing RPC health is different from P2P
  node health.
- The symptom sections map dashboard observations to concrete next checks.
- The checklist is role-neutral enough for validators, collators, RPC nodes,
  and test nodes.

## Editorial quality

- Headings are clear and scannable.
- Markdown formatting is consistent.
- The article avoids unsupported market, investment, or reward claims.
- The tone is technical and neutral.
