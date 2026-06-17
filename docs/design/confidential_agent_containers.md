# Confidential Agent Containers: Architecture and Design

Status: **Draft design proposal**

[TOC]

## Abstract

Confidential Containers (CoCo) protects a workload from an untrusted host and
uses remote attestation to decide whether secrets may enter the trusted
environment. An AI agent changes the workload model: it interprets untrusted
content, selects tools, creates sub-tasks, acquires credentials, and causes
externally visible side effects over a long-running session. Measuring the
container at boot is therefore necessary but not sufficient.

This document proposes **Confidential Agent Containers (CAC)**, an evolution of
CoCo that makes an agent's identity, authority, tool use, data flow, delegation,
and evidence auditable and enforceable. It retains CoCo's existing Kata Agent,
Attestation Service, and KBS resource policies. For portable agent lifecycle
governance it adopts Microsoft's open Agent Control Specification (ACS), with a
confidential-computing profile called the Confidential Agent Policy Standard
(CAPS), defined in
[Confidential Agent Policy Standard](confidential_agent_policy.md).

The central rule is:

> Attestation establishes what is running; authorization continuously limits
> what that measured agent may do for this principal, in this session, with
> this data and budget.

## Motivation

A conventional confidential workload has a relatively static image, input,
and secret set. An agent has a dynamic control loop:

1. receive goals and untrusted context;
2. plan and select a tool;
3. obtain narrowly scoped authority;
4. invoke a local or remote tool;
5. observe the result and update memory;
6. delegate or continue until a stop condition is reached.

This creates risks that a boot-time allowlist does not fully address:

* prompt injection can turn data into attempted authority;
* a correct model can choose an unsafe tool or unsafe arguments;
* credentials released once can be reused outside their intended action;
* a child agent can silently accumulate or amplify its parent's authority;
* model, adapter, system prompt, tool, and memory versions can change
  independently;
* network output can exfiltrate secrets through an otherwise allowed endpoint;
* an agent can consume unbounded money, tokens, time, or side effects;
* nondeterministic decisions are difficult to reconstruct after an incident;
* accelerator state, model weights, and KV cache may cross trust boundaries.

CAC treats every side effect as a policy-mediated action and every credential
as a short-lived, action-bound capability.

## Relationship to Confidential Containers

This proposal builds on the current CoCo architecture rather than replacing it:

* The **Kata Agent policy** remains the in-guest control over host-originated
  operations.
* The **Attestation Service (AS) policy** still appraises TEE evidence and
  reference values.
* The **KBS resource policy** still controls release of secrets and resources.
* **Trustee** remains the relying-party boundary containing KBS, AS, and the
  Reference Value Provider Service (RVPS).
* **Init-Data** remains the measured bootstrap channel for immutable workload
  configuration.

Current CoCo policy layering is documented in the
[CoCo policy overview](https://confidentialcontainers.org/docs/attestation/policies/),
and the Trustee component split is described in the
[attestation architecture](https://confidentialcontainers.org/docs/attestation/architecture/).
CAC adds agent-specific layers without asking an attestation policy to become a
general application authorization language.

## Relationship to Agent Control Specification

The
[Agent Control Specification](https://github.com/microsoft/agent-governance-toolkit/blob/main/policy-engine/spec/SPECIFICATION.md)
defines a portable runtime-governance contract: eight lifecycle intervention
points, a canonical policy input, annotator and policy dispatch, normalized
verdicts, transforms, approvals, and fail-closed host enforcement. CAC adopts
that contract rather than defining a parallel agent action and verdict model.

ACS intentionally treats its host as the stateful enforcement point. The host
assembles snapshots, tracks information-flow labels, executes dispatchers, and
acts on verdicts. In CAC, the measured APE is the ACS host. CAPS is therefore a
confidential-computing **profile of ACS** that signs and attests the resolved ACS
manifest, defines trusted snapshot claims, binds verdicts to KBS capabilities,
and adds rollback-resistant state and receipts. The agent framework adapter is
an untrusted event source, not the final enforcement boundary.

## Goals

CAC is designed to:

* bind an agent identity to attested code and configuration;
* enforce least authority at each tool call, not only at pod admission;
* prevent authority escalation through prompt text, memory, or delegation;
* release secrets just in time and bind them to an action where possible;
* support local tools, remote APIs, model endpoints, and child agents;
* produce privacy-preserving, tamper-evident decision and action receipts;
* work across TDX, SEV-SNP, CCA, Secure Execution, and future CoCo TEEs;
* preserve CoCo's separation between infrastructure operator and workload
  owner;
* allow a useful baseline implementation with existing CoCo components.

## Non-goals

CAC does not claim to:

* prove that a model's semantic output is correct or safe;
* make a TEE immune to side channels, denial of service, rollback, or hardware
  compromise;
* standardize model safety evaluation or alignment;
* expose chain-of-thought or require private prompts in audit logs;
* replace Kubernetes admission, network policy, seccomp, or supply-chain
  controls;
* make an external tool trustworthy merely because its caller is attested.

## Threat model

### Protected assets

* prompts, retrieved context, model weights, adapters, KV cache, and memory;
* user and service credentials;
* agent policy, system instructions, and tool schemas;
* integrity of actions, approvals, budgets, and audit records;
* tenant separation on CPUs, accelerators, storage, and model servers.

### Adversaries

CAC assumes any of the following may be malicious or compromised:

* cloud host, hypervisor-facing control plane, node administrator, or storage;
* Kubernetes administrator after workload policy has been committed;
* registry, network, retrieved document, web page, tool response, or user input;
* model-generated plan or child agent;
* remote tool service outside its explicitly attested and authorized contract.

The workload owner, configured trust roots, verifier, policy issuer, and the TEE
hardware root are trusted within their stated roles. Deployment profiles SHOULD
separate policy signing, reference-value administration, and infrastructure
operation.

### Required invariants

1. Untrusted content never grants authority.
2. A child receives no authority not held and explicitly delegated by its
   parent.
3. A secret is released only after fresh attestation and authorization for its
   intended use.
4. Policy cannot be weakened through a host-originated runtime request.
5. Security-relevant state has anti-rollback protection or fails closed when
   freshness cannot be established.
6. Every permitted side effect has a stable action identifier and a decision
   receipt.
7. Denial is the default for missing, malformed, expired, or unknown inputs.

## Architecture

```text
 Workload owner / principal                   Trustee
 ┌──────────────────────────┐     ┌─────────────────────────────────┐
 │ signed CAPS/ACS bundle   │     │ AS + RVPS: appraise TEE, image,│
 │ goals and approvals      │     │ Init-Data, model and policy    │
 └────────────┬─────────────┘     │ KBS: release action resources  │
              │                   │ Capability issuer / revocation │
              ▼                   └──────────────┬──────────────────┘
 ┌──────────────── confidential guest ──────────┼───────────────────┐
 │ Measured bootstrap / Kata Agent policy       │                   │
 │                                               ▼                   │
 │ ┌───────────────┐  proposed action   ┌───────────────────────┐  │
 │ │ Agent runtime │───────────────────►│ Agent Policy Enforcer │  │
 │ │ model/planner │◄───────────────────│ (APE)                 │  │
 │ └───────┬───────┘ decision/capability└──┬──────────┬─────────┘  │
 │         │                                │          │            │
 │         │                         ┌──────▼───┐ ┌────▼─────────┐  │
 │         └────────────────────────►│ Tool GW  │ │ Evidence log │  │
 │                                   └──────┬───┘ └──────────────┘  │
 └──────────────────────────────────────────┼────────────────────────┘
                                            ▼
                                  local/remote tools and agents
```

### Agent Policy Enforcer

The **Agent Policy Enforcer (APE)** is a small, measured, in-guest reference
monitor. The agent runtime cannot directly access tool sockets, workload
credentials, Trustee, persistent memory, or unrestricted egress. It submits a
structured lifecycle event to the APE through an ACS-compatible adapter. The
APE:

1. authenticates the calling process and session;
2. validates policy signatures, version, expiry, and monotonic epoch;
3. canonicalizes and validates the lifecycle event and selected policy target;
4. constructs the trusted ACS snapshot and evaluates the configured policy;
5. obtains any required remote authorization or human approval;
6. requests an action-bound resource or credential from Trustee;
7. invokes the tool gateway or returns a single-use capability;
8. records the decision and commits resulting state atomically.

The APE MUST be outside the model/plugin process and MUST mediate all paths to a
side effect. Linux isolation, separate UIDs, mount namespaces, seccomp, cgroups,
and explicit socket ownership SHOULD make bypass difficult even after arbitrary
code execution in the agent runtime.

### Packaging, deployment, and ownership

The APE is **not an ordinary Kubernetes sidecar in the preferred architecture**.
An ordinary sidecar is selected and replaceable through the pod specification,
usually shares the workload's lifecycle, and may be controlled by the untrusted
cluster control plane. Giving such a container authority over attestation,
credentials, and egress would weaken the CoCo trust boundary.

The preferred deployment is a **measured guest system service**:

* the CoCo platform or runtime vendor packages the APE binary and its minimal
  dependencies in the signed confidential guest image, alongside other
  guest-components;
* the guest init system starts the APE before any tenant workload container and
  keeps its lifecycle independent of the agent container;
* Kata Agent creates the workload container without granting it access to the
  APE's state, session key, policy store, Trustee channel, or gateway control
  plane;
* the workload communicates with the APE only through a narrow, authenticated
  guest-local API, such as a protected Unix domain socket;
* the guest firewall and namespace configuration deny direct workload egress,
  so stopping or bypassing the APE fails closed;
* the APE binary, configuration roots, and startup mode are covered by guest
  measurement and verified by Trustee before any agent credential is issued.

Responsibility is intentionally split:

| Responsibility | Owner |
| --- | --- |
| Build, patch, and publish the APE and confidential guest image | CoCo distribution, runtime vendor, or platform security team |
| Select the approved guest image/runtime class on a cluster | Cluster confidential-computing platform operator |
| Author and sign the CAPS envelope, ACS manifest/policies, budgets, and data rules | Workload owner or tenant security authority |
| Register expected APE/guest measurements and policy issuer roots | Tenant trust administrator through RVPS/Trustee |
| Start and supervise the measured APE | Guest init/runtime, not the Kubernetes control plane |
| Appraise the APE and release its session authority | Trustee operated by, or accountable to, the tenant trust domain |
| Submit proposed actions | Unprivileged agent workload |

Thus the platform operator **installs the mechanism**, while the workload owner
**supplies the authority policy**. Neither party alone should silently gain both
roles: a platform operator cannot weaken tenant policy without changing the
attested measurement, and a workload owner cannot replace the measured
enforcer merely by changing a pod manifest.

A prototype MAY package the APE as a dedicated container inside the confidential
pod sandbox. This is called the **guest service-container profile**, not a
standard sidecar profile. It is conforming only if the guest runtime, rather
than the untrusted host, pins its digest and launch configuration in measured
Init-Data; launches it before the agent; gives it a distinct identity and
namespaces; prevents the agent and pod API from replacing or stopping it; and
includes it in attestation appraisal. A normal user-declared sidecar that can be
changed with `spec.containers` MUST NOT be trusted as the CAPS/ACS enforcer.

An application-linked APE library is suitable only for a lower-assurance
single-process profile. It cannot claim protection against compromise of the
agent runtime because the policy enforcer and its caller share a failure domain.

APE upgrades require a new signed guest image or measured service-container
digest, updated RVPS reference values, and a fresh attestation/session. They are
never performed as an unmeasured hot replacement.

### Tool Gateway

The Tool Gateway is the only default egress path. It enforces destination,
method, protocol, request schema, response size, timeout, rate, and data-label
rules. High-assurance profiles SHOULD terminate application TLS inside the
guest and authenticate tools with short-lived workload identities.

A remote tool SHOULD publish a signed tool descriptor containing its identity,
input/output schema digests, side-effect class, data-use claims, and optional
attestation requirements. A digest of that descriptor is authorized by the
signed CAPS envelope and ACS policy;
the model-provided tool description is never authoritative.

### Attested Agent Identity

After successful attestation, an Agent Identity Service issues a short-lived
identity bound to:

* TEE and runtime appraisal result;
* guest image and Init-Data measurements;
* agent runtime, model, tokenizer, adapters, and system-prompt digests;
* CAPS envelope, resolved ACS manifest, and policy epoch digests;
* tenant, workload, session, and public key generated inside the TEE.

A deployment MAY encode the identity as an X.509-SVID or another proof-of-
possession credential. SPIFFE is a useful compatibility profile because its
workload identity and short-lived credential model is already portable; see the
[SPIFFE concepts](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/).
An identity credential does not itself grant tool authority.

### Capability Broker

The Capability Broker extends the KBS role from releasing long-lived bytes to
issuing constrained authority. It accepts attestation results plus an authorized
action and returns one of:

* a secret over the attestation-bound secure channel;
* a single-use signed capability token;
* a short-lived mTLS certificate;
* a cloud/KMS token restricted by audience and provider policy;
* a cryptographic operation handle, keeping key material non-exportable.

Capabilities MUST include audience, action digest, policy digest, session,
expiry, and unique token identifier. They SHOULD include a maximum use count of
one. Tools MUST reject a token when the request does not match its bound action.
Bearer credentials SHOULD NOT be used when proof-of-possession is available.

### Evidence and Receipt Log

The APE maintains an append-only hash chain rooted in a TEE-held key. Each entry
contains metadata and digests, not raw prompts or secrets by default. A receipt
covers:

* previous receipt hash, sequence number, and session;
* policy, model, tool descriptor, and canonical action digests;
* decision, matched rule, obligations, approval references, and budget delta;
* result class and digest;
* wall-clock time when trusted, plus monotonic ordering always;
* APE signature and optional transparency-log inclusion proof.

Receipts support incident response and billing without exposing chain-of-
thought. A verifier can detect deletion of a suffix only when checkpoints are
periodically anchored outside the guest, so production deployments SHOULD
anchor receipt roots to a tenant-controlled transparency or audit service.

## Trust and policy planes

CAC uses five deliberately separate policy planes:

| Plane | Enforcer | Question answered |
| --- | --- | --- |
| Guest lifecycle | Kata Agent | May the untrusted host ask the guest to perform this lifecycle operation? |
| Attestation | AS | Is this TEE/runtime state acceptable and what claims describe it? |
| Resource release | KBS | May this attested workload obtain this resource? |
| Agent action | APE | May this agent perform this canonical tool action now? |
| Tool acceptance | Tool/service | Is this capability valid for this exact request and caller? |

The same signed policy bundle MAY generate inputs for several planes, but each
plane has an independent deny-by-default decision. An allow at one plane MUST
NOT imply an allow at another.

## Lifecycle

### Build and registration

1. Produce signed OCI artifacts for guest image, agent runtime, model,
   tokenizer, adapters, tools, and policy bundle.
2. Register expected measurements and artifact digests with RVPS.
3. Compile the workload's Kubernetes/OCI description into a restrictive Kata
   Agent policy.
4. Resolve and validate the ACS manifest, sign its CAPS envelope, and place the
   envelope digest and issuer roots in Init-Data.
5. Configure KBS resources against stable logical names, never model-generated
   resource paths.

OCI descriptors already provide content digests and namespaced annotations;
CAC uses those primitives rather than inventing mutable image identity. See the
[OCI descriptor](https://specs.opencontainers.org/image-spec/descriptor/) and
[annotation](https://specs.opencontainers.org/image-spec/annotations/)
specifications.

### Boot and attestation

1. The guest boots and measures Init-Data and the APE.
2. The APE generates an ephemeral session key in the TEE.
3. Attestation evidence binds the key and a fresh verifier challenge.
4. AS appraises evidence against RVPS reference values and emits attestation
   results.
5. Trustee verifies the CAPS envelope, ACS manifest digest, and epoch, then
   issues a short-lived agent identity.
6. Tool egress remains closed until bootstrap completes successfully.

This follows the RATS separation of Attester, Verifier, and Relying Party in
[RFC 9334](https://www.rfc-editor.org/rfc/rfc9334.html). An implementation MAY
encode interoperable attestation claims using EAT
([RFC 9711](https://www.rfc-editor.org/rfc/rfc9711.html)).

### Action execution

1. The agent adapter proposes an ACS lifecycle event, such as `pre_tool_call`.
2. The APE rejects attempts to populate trusted fields, adds attested identity,
   labels, budget, approval, and session state, and builds the ACS snapshot.
3. The pinned ACS runtime projects the policy target, runs configured
   annotators and policy dispatchers, and returns a normalized verdict.
4. The APE enforces `allow`, `warn`, `deny`, `escalate`, or `transform` at the
   actual side-effect boundary; it never delegates final enforcement to the
   agent adapter.
5. For an allowed side effect, the APE reserves budget and obtains an
   event-bound capability from Trustee.
6. The gateway invokes the tool using the transformed target when applicable
   and verifies that the response matches limits.
7. The APE evaluates `post_tool_call` and commits the budget/result receipt.
   Ambiguous outcomes are not automatically retried for non-idempotent actions.

### Delegation

A child agent receives a signed delegation whose authority is the intersection
of:

* the parent's remaining authority;
* a policy rule allowing delegation to the child identity and purpose;
* the explicitly delegated tools, data labels, budget, and expiry.

The child MUST receive a new session key and identity. Delegation depth,
fan-out, cumulative budget, and expiry MUST decrease monotonically. A child
cannot delegate unless the parent delegation explicitly permits it.

### Update and revocation

Policies and models are versioned independently. A policy update increments a
monotonic epoch. The APE accepts only equal or greater epochs signed by an
allowed issuer; rollback causes the session to fail closed. Emergency
revocation can deny new capabilities immediately. Existing capabilities must be
short-lived because offline revocation cannot be guaranteed.

A model, adapter, system prompt, tool descriptor, or APE update changes the
attested agent identity and requires re-authorization. Live migration and
checkpoint restore require a fresh session identity and proof that sealed state
has not rolled back.

## Accelerator and model-serving profiles

### In-guest inference

The strongest profile places model execution and policy enforcement in the same
confidential guest. Accelerator assignment MUST prevent host or peer DMA into
private memory. Accelerator firmware, driver, and device evidence SHOULD be
composed into the attestation result. Before release or reassignment, GPU memory,
KV cache, and staging buffers MUST be scrubbed.

### Attested remote model service

A smaller agent guest may call a separately attested model service such as a
confidential vLLM deployment. Mutual attestation binds both identities. The
CAPS envelope and ACS policy specify acceptable model-service identity, model/adapter digests, retention
class, and maximum prompt/output labels. The model service receives no agent
credentials and cannot call tools directly.

### Non-confidential model service

This profile MUST reject confidential prompts and secrets by default. Policy
may allow public or explicitly declassified data. Redaction is an obligation,
not a claim inferred from the model's intent.

## Kubernetes integration

A future `ConfidentialAgent` custom resource can reference immutable artifacts
and policy without embedding secrets:

```yaml
apiVersion: confidentialcontainers.org/v1alpha1
kind: ConfidentialAgent
metadata:
  name: invoice-reconciler
spec:
  runtimeClassName: kata-cc
  serviceAccountName: invoice-agent
  agentImage: registry.example/agent@sha256:...
  model: registry.example/models/model@sha256:...
  policyBundle: registry.example/policies/invoice@sha256:...
  policyEpoch: 42
  trustDomain: agents.example
  isolationProfile: dedicated-confidential-vm
  receiptSink: https://audit.example/v1/roots
```

Admission MUST resolve tags to digests, reject mutable policy references, bind
the service account and namespace into Init-Data, and select a runtime class
that satisfies the policy's TEE profile. The `ConfidentialAgent` resource does
not deploy the APE as another entry in `spec.containers`; it selects a
confidential guest image/runtime class that already contains the measured APE.
The workload owner supplies the signed `policyBundle`, while the platform
operator supplies the approved enforcement-capable runtime. Kubernetes
admission is a convenience; Trustee and the in-guest APE remain authoritative
against a malicious control plane.

## Failure semantics

* Policy engine unavailable: deny new actions.
* Trustee unavailable: continue only already-authorized local, non-side-effect
  actions whose policy explicitly permits offline operation.
* Audit anchor unavailable: buffer up to a policy limit, then deny side effects.
* Trusted time unavailable: use verifier freshness and short session leases;
  deny after lease exhaustion.
* Budget store conflict: deny and reconcile; never double-spend.
* Unknown tool response: label untrusted, do not execute embedded instructions.
* Crash after dispatch: record `outcome_unknown`; retry only with a stable
  idempotency key and a tool contract that guarantees deduplication.

## Implementation roadmap

### Phase 0: policy bundle and gateway

* Define the CAPS envelope schema, ACS compatibility matrix, and snapshot
  conventions.
* Add the APE as a measured, guest-init-managed system service.
* Generate restrictive Kata Agent policy from the same bundle.
* Route all network and tool access through a UDS-connected gateway.
* Emit signed local receipts.

### Phase 1: Trustee integration

* Add policy/model/tool digests to attestation and KBS policy inputs.
* Introduce `AuthorizeAction` and single-use capability issuance.
* Bind resource release to action digest and session public key.
* Anchor receipt roots externally and add revocation epochs.

### Phase 2: portable identity and delegation

* Issue attestation-bound workload identities.
* Add child-agent delegation and budget intersection.
* Standardize remote tool descriptors and tool-side token verification.
* Support human approval services with signed approval receipts.

### Phase 3: composed accelerator attestation

* Bind accelerator evidence, firmware, driver, and model identity.
* Define secure checkpoint, migration, and device-scrub profiles.
* Add conformance tests across CoCo-supported TEEs.

## Security considerations

* **Confidentiality is not authorization.** A TEE can faithfully run a dangerous
  agent; ACS policy restricts its behavior while CAPS makes the host and claims
  attestable.
* **Measurement is not semantic identity.** Agent identity includes the model,
  prompt/configuration, policy, and tools, not only the guest image.
* **Prompt text is never policy.** The APE parses only typed envelopes produced
  through a trusted API and ignores attempted credentials or policy statements
  in untrusted fields.
* **Logs can leak.** Receipts contain digests and policy metadata by default;
  optional content capture requires explicit retention and access policy.
* **Policy engines are attack surface.** Implementations should use a small,
  deterministic evaluator, bounded input sizes, fuel/time limits, and no
  evaluator network access.
* **Side channels remain.** Deployments must document TEE, accelerator,
  co-tenancy, traffic-analysis, and denial-of-service residual risks.

## Open questions

* Which action claims should become a CoCo-specific EAT profile versus a
  separate capability token?
* Should the preferred measured guest service be delivered as a guest-components
  daemon or a separately packaged service, while preserving the same lifecycle
  and isolation contract?
* How should an accelerator's evidence be composed across vendors without
  making KBS hardware-specific?
* Which tool-side capability format gives the best interoperability while
  retaining proof-of-possession and request binding?
* How should policy authors express semantic constraints on large arguments
  without making policy evaluation model-dependent?
