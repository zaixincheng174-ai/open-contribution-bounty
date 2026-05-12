# Polkadot Telemetry: Monitoring and Analyzing Network Health

Running a Polkadot or Polkadot SDK node is not finished when the process
starts. Operators need to know whether the node is syncing, keeping peers,
importing blocks, finalizing, exposing the right endpoints, and staying healthy
under real network conditions.

Polkadot Telemetry is one of the fastest ways to see that high-level network
health picture. It gives operators and ecosystem observers a shared view of
nodes, block progress, implementation versions, peer activity, and basic
resource signals. It should not be treated as a complete monitoring system, but
it is a useful first dashboard for answering a simple question: is this node
alive and keeping up with the network?

This article explains what Telemetry shows, how it fits beside logs and
Prometheus metrics, what signals matter during operations, and how to use it
without over-trusting it.

## What Polkadot Telemetry is

Polkadot Telemetry is a public dashboard backed by a telemetry ingestion
service. Nodes can send lightweight status messages to a telemetry backend, and
the frontend displays those messages in a searchable interface.

The default public dashboard is useful for checking public networks such as
Polkadot, Kusama, and test networks. Teams can also run their own telemetry
backend for private networks, test environments, or internal dashboards.

Telemetry is most useful for quick visibility:

- Is the node visible on the expected network?
- Is it syncing or fully synced?
- What best block and finalized block does it report?
- How many peers does it currently have?
- Which node version and implementation is it running?
- Is the node falling behind other nodes?
- Are many nodes on the same network showing similar symptoms?

It is less useful for deep incident response by itself. For that, operators
still need local logs, Prometheus metrics, host metrics, alerting, and access to
the node or infrastructure.

## How a node sends telemetry

Polkadot SDK nodes can be configured with a telemetry endpoint through the
`--telemetry-url` flag. The telemetry URL includes both the WebSocket endpoint
and a verbosity level as one quoted argument.

For a local telemetry backend, the Substrate Telemetry repository shows the
shape of the argument:

```sh
polkadot --dev --telemetry-url 'ws://localhost:8001/submit 0'
```

The final `0` is the telemetry verbosity level, not part of the WebSocket URL.
Verbosity level `0` is enough for most dashboard information such as connection
status, interval updates, block import, and finalized block notifications.
Higher verbosity can send extra consensus-related data, but it is usually not
needed for a normal health dashboard.

For production nodes, operators should decide whether the public dashboard is
appropriate. Validators, sentry nodes, collators, RPC nodes, and private
infrastructure may have different privacy and exposure requirements. Telemetry
is useful, but publishing too much operational metadata can make an operator's
infrastructure easier to fingerprint.

## What the dashboard can tell you

Telemetry is a fast way to compare one node against the rest of the network.
The exact UI can change, but the operational questions are stable.

### Sync status

A new node should make steady progress toward the network's current best block.
During sync, it may appear behind the rest of the network and may have lower
peer stability. That is normal if the best block keeps increasing.

Warning signs include:

- The node stays at the same block for a long time.
- Best block increases but finalized block does not move.
- The node repeatedly disappears and reappears.
- The node reports a different chain than expected.
- The node is far behind peers with similar infrastructure.

Sync status should be confirmed with local logs and metrics before making a
production decision. Telemetry can show the symptom, but it cannot always show
the cause.

### Peer count and peer quality

Peer count is one of the most visible network-health indicators, but it is easy
to misuse. A node with zero peers or very few peers is probably misconfigured,
firewalled, pointed at the wrong chain, or unable to reach bootnodes. A node
with many peers is not automatically healthy.

Peer quality matters more than raw count. Operators should look for stable
connectivity, steady syncing, block import progress, and low churn. A node can
have many connections and still be unhealthy if it is overloaded, lagging, or
connected to poor peers.

### Version and client distribution

Telemetry can help operators spot version drift. If most network participants
are running a newer client and a few nodes remain on an old build, those older
nodes deserve attention before a runtime upgrade, client upgrade, or operational
incident.

Version information is also useful during rollouts. Operators can check whether
their own nodes are reporting the expected binary and whether the broader
network is converging on a compatible release.

### Node role and identity

Telemetry can show node names and identifiers that help operators find their
own infrastructure. This is useful for test networks, bootnodes, validators,
collators, and public RPC nodes.

Node naming should be intentional. A clear node name helps operations, but it
should not expose secrets, private hostnames, customer names, private IP
patterns, or internal incident details.

## Telemetry, logs, Prometheus, and RPC each answer different questions

A reliable monitoring setup does not rely on a single signal. Telemetry,
Prometheus, logs, and RPC checks are complementary.

Telemetry answers:

- Is the node visible to the telemetry backend?
- What high-level status does the node report?
- How does this node compare with other nodes on the network?

Logs answer:

- What exactly is the node doing locally?
- Are there repeated import, database, network, consensus, or RPC errors?
- Did the node reject peers or fail to bind a port?

Prometheus metrics answer:

- What are the time-series trends for block height, peers, CPU, memory, disk,
  network traffic, queues, and process health?
- Did a metric cross an alert threshold?
- Did resource pressure begin before network symptoms appeared?

RPC and chain observation answer:

- Can clients query the node successfully?
- Does the node serve the expected finalized state?
- Are application-facing methods available and responsive?

Treating these signals separately helps avoid false conclusions. For example,
a node can appear on Telemetry while its public RPC endpoint is broken. A node
can answer local RPC while having poor peer connectivity. A node can show a
healthy process while disk I/O is already causing slow imports.

## A practical monitoring baseline

For a production node, a minimal baseline should include more than Telemetry.

Useful checks include:

- Process supervision through systemd, containers, or an orchestration layer.
- Local logs with retention and searchable incident history.
- Prometheus metrics exposed only to trusted monitoring infrastructure.
- Host metrics for CPU, memory, disk I/O, disk capacity, and network traffic.
- Import and queue signals that show whether the node is receiving work faster
  than it can process blocks.
- Alerts for stalled best block, stalled finalized block, low peer count,
  process restarts, disk pressure, and RPC error rate.
- External checks for public RPC or load-balanced endpoints, if the node serves
  users.
- Runbooks for upgrade, rollback, database recovery, and peer-connectivity
  incidents.

The exact alert thresholds should be network-specific. A validator, a collator,
an archive RPC node, and a temporary test node do not need the same policy.

## Alert and dashboard examples

Prometheus and Grafana should turn the baseline into repeatable operator
signals. The exact metric names depend on the node version, exporter, and host
monitoring stack, so teams should verify names in their own `/metrics` output
instead of copying example expressions blindly.

Good starting alerts include:

- Best block has not increased for several minutes while comparable peers keep
  moving.
- Finalized block has stopped advancing while best block still imports.
- Peer count stays below the node role's minimum operating range.
- Process restarts exceed the expected deployment or maintenance pattern.
- Disk capacity, disk I/O wait, memory pressure, or CPU saturation crosses the
  host team's operational threshold.
- Public RPC error rate or latency rises while Telemetry still looks healthy.

A useful Grafana dashboard should separate chain health from host and user
traffic. Put block height, finalized height, and peer count together; keep CPU,
memory, disk, and network panels in a host section; and put RPC request rate,
latency, and error rate in an application-facing section. This layout helps an
operator see whether an incident starts in the chain, the machine, or the
public endpoint path.

## Upgrade and resource-pressure watchpoints

Runtime upgrades, client releases, and infrastructure migrations deserve closer
monitoring than a normal steady-state period. After a change, operators should
watch whether best block, finalized block, peer count, import queues, database
activity, and host resource metrics continue to move together.

A node that keeps receiving blocks but imports them slowly may look partially
alive while it falls behind the network. That pattern usually points to local
resource pressure, slow storage, database work, or an overloaded node role. The
fastest response is to compare Telemetry with local metrics and logs before
assuming a network-wide incident.

## Reading common symptoms

### The node is not visible on Telemetry

Possible causes:

- The node was not started with a telemetry URL.
- The telemetry URL or verbosity argument is malformed.
- Outbound WebSocket traffic is blocked.
- The node is on a private network or different chain.
- The telemetry backend is unavailable.

First check local logs and the startup command. If the node is otherwise
healthy, invisibility on Telemetry may be a telemetry configuration problem,
not a chain-health problem.

### The node has low or zero peers

Possible causes:

- P2P ports are blocked by firewall or cloud security rules.
- Bootnodes or reserved nodes are unreachable.
- The node is using the wrong chain specification.
- The node is behind NAT without suitable connectivity.
- The binary version is incompatible with peers.

Check listening addresses, reserved peer configuration, bootnode reachability,
and logs from the networking subsystem.

### Best block moves but finalized block stalls

This can happen briefly, but a prolonged gap deserves attention. It may be a
network-wide issue, a local import issue, or a sign that the node is not keeping
up with finality-related messages.

Compare the node against other nodes on Telemetry and confirm with local logs.
If many nodes show the same pattern, it may be broader than one operator's
machine.

### RPC users report problems while Telemetry looks healthy

Telemetry does not prove RPC health. Check RPC listener configuration, reverse
proxy health, rate limits, method restrictions, load balancer status, and client
error logs.

This distinction is important for public infrastructure. A node can stay synced
and visible while the user-facing API path is overloaded or misconfigured.

## Security and privacy considerations

Telemetry is an observability tool, not a secret-management system. Operators
should avoid leaking sensitive details through node names, locations, hostnames,
or patterns that expose the full infrastructure layout.

For validators, think carefully before exposing direct validator identity and
topology. A sentry-node design may intentionally make sentries public while
keeping validator machines less visible. Internal monitoring can be richer than
public telemetry.

Prometheus and RPC endpoints also need protection. Exposing metrics or unsafe
RPC methods to the public internet can create operational and security risk.
Public dashboards are useful, but administrative interfaces should stay behind
firewalls, VPNs, authentication, or private networking.

## Operator checklist

Before relying on a node, confirm:

- The node is visible on the expected network, if public telemetry is enabled.
- The node name is useful but does not leak sensitive information.
- Best and finalized blocks move consistently.
- Peer count is stable enough for the node role.
- The reported version is the intended release.
- Local logs do not show repeated import, network, database, or RPC errors.
- Prometheus metrics are collected by trusted infrastructure.
- Alerts cover stalled sync, low peers, process restarts, disk pressure, and
  RPC health where relevant.
- Public RPC endpoints are monitored separately from P2P node health.
- Upgrade and rollback runbooks exist before a critical runtime or client
  release.

## Summary

Polkadot Telemetry gives operators a fast, shared view of network health. It is
best used as an early signal and comparison tool: it shows whether a node is
visible, syncing, finalizing, connected to peers, and running the expected
version.

Strong operations require more than that. Telemetry should be combined with
logs, Prometheus metrics, host monitoring, RPC checks, alerting, and runbooks.
When these layers agree, operators can diagnose incidents faster and avoid
mistaking a single dashboard signal for complete node health.

## References

- [Polkadot Telemetry Dashboard](https://telemetry.polkadot.io/)
- [Substrate Telemetry Repository](https://github.com/paritytech/substrate-telemetry)
- [Polkadot Developer Docs: Set Up a Node](https://docs.polkadot.com/infrastructure/running-a-node/setup-full-node)
- [Polkadot Developer Docs: Run an RPC Node](https://docs.polkadot.com/infrastructure/running-a-node/)
- [Polkadot Developer Docs: Start Validating](https://docs.polkadot.com/node-infrastructure/run-a-validator/onboarding-and-offboarding/start-validating/)
- [Prometheus Documentation: Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation: Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
