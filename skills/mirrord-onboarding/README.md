# mirrord-onboarding

Onboard a repo onto mirrord end to end: pick the use cases, make the session header propagate, decide how much database state sessions share, write a mirrord config per service, and wire up remocal, CI, and preview environments.

## What it does

This skill helps AI agents:
- **Ask** which use cases to cover: remocal (local code against the shared cluster), CI, and per-PR preview environments
- **Propagate** the `baggage` session header across services and queues, via the `mirrord-header-propagation` skill
- **Ask** how much each database should be shared with the environment, and set up DB branching to match
- **Write** a checked-in `.mirrord/mirrord.json` per service, with session-keyed HTTP filters, queue splitting, and DB branches
- **Implement** each chosen use case: run configs / `mirrord-up.yaml`, a `mirrord ci` job, and a preview workflow
- **Report** what was set up per service, what's left for the user to apply, and which flows aren't isolated

## Example prompts

```
"Set up mirrord for our repo"

"Onboard our services to mirrord for local dev, CI, and per-PR previews"

"Roll out mirrord across the team on our staging cluster"

"Add mirrord configs for every service and a preview environment per PR"
```

## Prerequisites

- Source access to the services and their Kubernetes manifests
- A shared dev/staging cluster. Session filters, queue splitting, DB branching, CI, and previews need the mirrord Operator (Team / Enterprise; CI API keys and previews are Enterprise). The skill checks and points to the `mirrord-operator` skill if it's missing

## Learn more

- [mirrord documentation](https://metalbear.com/mirrord/docs)
- [Sharing the cluster](https://metalbear.com/mirrord/docs/sharing-the-cluster)
