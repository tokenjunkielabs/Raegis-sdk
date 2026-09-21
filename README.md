#  Raegis SDK

The official TypeScript SDK for the **Raegis RWA Protocol**. This library provides a clean, class-based interface to interact with Raegis Soroban smart contracts on the Stellar network.

##  Installation

```bash
npm install @aegis/sdk
```

## Quickstart

Start with the least privilege your application needs. Read-only integrations
do **not** need a Stellar secret key. Keep signer material outside source code
and only construct a signer-capable client for a flow that must submit a
transaction.

### Environment

Keep public configuration separate from signer secrets:

```bash
# Public configuration
AEGIS_ENVIRONMENT=testnet
AEGIS_CONTRACT_ID=C_YOUR_CONTRACT_ID

# Optional only when using explicit AegisClient network configuration.
# Use a complete URL including the https:// scheme.
AEGIS_RPC_URL=https://soroban-testnet.stellar.org

# Private server-side secret for privileged examples only.
# Never commit this value or expose it in a browser/mobile bundle.
AEGIS_ADMIN_SECRET=S_YOUR_ADMIN_SECRET
```

For production, load signer secrets from your deployment platform's secret
manager. Do not place Stellar secret keys in committed `.env` files, example
fixtures, logs, client-side JavaScript, or mobile application bundles.

### Read-only compliance query

A read-only client accepts no keypair. It is suitable for dashboards, indexers,
and compliance checks that only query contract state.

```typescript
import { createReadOnlyClient } from '@aegis/sdk';

const contractId = process.env.AEGIS_CONTRACT_ID;
if (!contractId) {
  throw new Error('AEGIS_CONTRACT_ID is required');
}

const reader = createReadOnlyClient({
  environment: 'testnet',
  contractId,
});

const isApproved = await reader.compliance.checkWhitelist(
  'G_USER_PUBLIC_KEY',
);
console.log({ isApproved });
```

### Privileged admin operation

Signer-capable clients require a `Keypair`. Load the secret at runtime from a
trusted server-side secret source and create the signer only for the privileged
operation that needs it. The SDK role guard is a client-side sanity check; the
contract remains the authorization authority.

```typescript
import { createAdminClient } from '@aegis/sdk';
import { Keypair } from '@stellar/stellar-sdk';

const contractId = process.env.AEGIS_CONTRACT_ID;
const adminSecret = process.env.AEGIS_ADMIN_SECRET;

if (!contractId || !adminSecret) {
  throw new Error('AEGIS_CONTRACT_ID and AEGIS_ADMIN_SECRET are required');
}

const admin = createAdminClient({
  environment: 'testnet',
  contractId,
  keypair: Keypair.fromSecret(adminSecret),
});

admin.assertAdminAccess();
const transactionHash = await admin.asset.mint(
  'G_INVESTOR_PUBLIC_KEY',
  10_000,
);
console.log({ transactionHash });
```

### Explicit RPC configuration

Prefer an `environment` preset when it fits your deployment. Advanced/custom
setups may construct `AegisClient` with an explicit RPC URL and network
passphrase. The RPC URL must be a complete URL such as
`https://soroban-testnet.stellar.org`, not a bare hostname.

```typescript
import { AegisClient } from '@aegis/sdk';
import { Networks } from '@stellar/stellar-sdk';

const contractId = process.env.AEGIS_CONTRACT_ID;
const rpcUrl = process.env.AEGIS_RPC_URL;

if (!contractId || !rpcUrl) {
  throw new Error('AEGIS_CONTRACT_ID and AEGIS_RPC_URL are required');
}

const reader = new AegisClient({
  rpcUrl,
  networkPassphrase: Networks.TESTNET,
  contractId,
});

// No keypair is configured, so this client is read-only.
const isApproved = await reader.compliance.checkWhitelist(
  'G_USER_PUBLIC_KEY',
);
console.log({ isApproved });
```

See [Role-Aware Client Factory](./docs/role-aware-client-factory.md) for the
full capability matrix, `compliance-operator` usage, signer requirements,
error handling, and security notes.

## Role Discovery & Capability Checks
Check what an address is classified as, and what it can currently attempt through the SDK.
This is a client-side convenience for UI gating, not on-chain authorization — see the
[full documentation](./docs/role-discovery.md) for important caveats.
```typescript
const roleResult = await aegis.role.discoverRole('G_USER_PUBLIC_KEY');
console.log('Role:', roleResult.role); // 'investor' | 'unauthorized' | 'unknown'

const capability = await aegis.role.checkCapability('G_USER_PUBLIC_KEY', 'receive_transfer');
console.log('Can receive transfer?', capability.isPermitted);
```

## Contract Event Decoder
Decode Soroban contract events into typed audit-trail models for dashboards and indexers.

```typescript
import { decodeContractEvent } from '@aegis/sdk';

const event = decodeContractEvent({
  topic: rpcEvent.topic,
  value: rpcEvent.value,
  txHash: rpcEvent.txHash,
});

if (event.kind === 'transfer') {
  console.log(event.from, event.to, event.amount);
}
```

See [Contract Event Decoder](./docs/contract-events.md) for supported topics, unknown fallback behaviour, and dashboard integration guidance.

## Testing
To run the SDK unit tests locally:

```
npm run test
```

Run the full release gate, including TypeScript compilation and browser/Node
runtime compatibility checks:

```bash
npm run check
```

### Pre-submit verification

Run all checks (lint, format, build, test, compat) in a single command before
submitting a PR:

```bash
npm run verify
```

See [Test-First Contribution Guide](docs/test-first-contribution.md) for when behavior changes need happy-path, negative-path, and no-test justification coverage.

See [Verification Command](docs/verification.md) for detailed usage and
troubleshooting guidance.

See [Runtime Compatibility](docs/runtime-compatibility.md) for the supported
environments, what the automated probes cover, and integration guidance.

For step-by-step instructions on reproducing and fixing CI check failures, see the [CI Resolution Workflow](docs/ci-resolution-workflow.md).

## Contributing
We welcome contributions! Please check our [CONTRIBUTING.md](CONTRIBUTING.md) for our branching strategy and code style guidelines.

Before submitting a PR, follow our [Test-First Contribution Guide](docs/test-first-contribution-guide.md) to understand when tests are required, what type of tests are expected per module, and how to prove your change works correctly.

Please also review our [Low-Effort PR Examples](docs/low-effort-pr-examples.md) to understand the quality standards for accepted contributions and to see examples of what to avoid (e.g., superficial changes, partial implementations, and untested code).

### Review Process
PRs submitted to this repository are reviewed against our [Pull Request Reviewer Checklist](docs/reviewer-checklist.md), which covers code implementation, unit test coverage, CI build compatibility, API reference documentation, security/compliance, and acceptance criteria.

### Acceptance Criteria Traceability
Every PR **must** include an [acceptance criteria traceability table](docs/acceptance-criteria-traceability.md) that maps SDK modules, tests, docs, and behaviour verification to each acceptance criterion from the linked issue. This makes evaluation straightforward for maintainers and GrantFox reviewers.

