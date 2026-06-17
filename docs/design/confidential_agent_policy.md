# Confidential Agent Policy Standard: ACS Confidential Profile 0.1

Status: **Draft proposed profile**

[TOC]

## Abstract

The Confidential Agent Policy Standard (CAPS) is a confidential-computing
profile of Microsoft's open
[Agent Control Specification (ACS)](https://github.com/microsoft/agent-governance-toolkit/blob/main/policy-engine/spec/SPECIFICATION.md).
It does not define another agent lifecycle, policy input, policy language, or
verdict model. ACS defines those portable runtime-governance semantics. CAPS
specifies what an ACS host MUST do when that host is an attested policy
reference monitor inside a Confidential Containers guest.

CAPS adds the properties that are outside the ACS runtime contract:

* a signed, immutable envelope around the resolved ACS manifest;
* binding of the ACS host, manifest, agent, model, and configuration to remote
  attestation;
* construction of trusted snapshot fields independently of the agent;
* complete mediation by a measured Agent Policy Enforcer (APE);
* action-bound KBS resources and proof-of-possession capabilities;
* rollback-resistant budgets, approvals, delegation, and session state;
* signed, hash-chained receipts anchored outside the guest.

The resulting split is deliberate:

> ACS defines where and how agent policy is evaluated. CAPS defines why the
> evaluator, its inputs, its state, and its enforcement can be trusted.

This profile targets ACS `0.3.1-beta`. ACS is a draft and may change. A CAPS
implementation MUST pin the exact supported ACS version and schema digest; it
MUST NOT silently reinterpret a manifest written for another version.

## Normative language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** are interpreted as described by
[BCP 14](https://www.rfc-editor.org/rfc/rfc2119.html) when, and only when, they
appear in all capitals.

## Relationship to ACS

ACS defines:

* the eight intervention points `agent_startup`, `input`, `pre_model_call`,
  `post_model_call`, `pre_tool_call`, `post_tool_call`, `output`, and
  `agent_shutdown`;
* the canonical policy input containing `intervention_point`, `policy_target`,
  `snapshot`, `annotations`, and `tool`;
* manifest projection paths, tool catalogs, annotators, policy dispatchers,
  approval resolvers, and manifest composition;
* normalized `allow`, `warn`, `deny`, `escalate`, and `transform` verdicts;
* deterministic, stateless, fail-closed evaluation;
* the host integration boundary that acts on the verdict.

The ACS specification explicitly makes the host responsible for assembling the
snapshot, tracking information-flow provenance and labels between evaluations,
executing annotators and policies, and enforcing verdicts. Those are ordinary
application trust assumptions in ACS. They become the primary security boundary
in a confidential agent. CAPS therefore profiles the **ACS host** as the APE.

CAPS does not fork ACS. Unless this document explicitly narrows an ACS option,
a conforming implementation follows the pinned ACS specification. CAPS fields
are not added to the ACS manifest top level because the ACS schema rejects
unknown top-level properties. They live in a signed outer envelope and in
reserved snapshot namespaces.

Microsoft's
[ACS introduction](https://commandline.microsoft.com/agent-control-specification-runtime-governance/)
positions ACS as framework-, runtime-, and policy-engine-independent lifecycle
governance. The Agent Governance Toolkit documentation also states that its
normal enforcement is application middleware and may share a process boundary
with the agent. CAPS keeps the ACS contract but moves its host and final I/O
enforcement into a separately measured guest service.

| Concern | ACS responsibility | CAPS confidential extension |
| --- | --- | --- |
| Lifecycle | Defines eight intervention points. | Requires the applicable points and prevents bypass around them. |
| Input | Defines canonical policy input assembled from a host snapshot. | Defines trusted namespaces and attested sources for snapshot claims. |
| Decision | Normalizes policy-engine results into ACS verdicts. | Binds verdict/input digests to capabilities and receipts. |
| State | Runtime is stateless; host carries session context. | APE protects budget, delegation, approval, and anti-replay state. |
| Enforcement | Host adapter acts on verdicts. | Measured APE owns actual model, tool, storage, and egress paths. |
| Policy loading | Validates ACS manifest and optional composition. | Signs, digest-pins, pre-resolves, measures, and anti-rolls back policy. |
| Evidence | Host dispatches annotators. | Assigns provenance/trust classes and constrains positive authority. |
| Identity/secrets | Outside the ACS runtime contract. | CoCo attestation and Trustee/KBS issue event-bound authority. |
| Audit | Host records evaluation results. | Attestation-bound, hash-chained receipts are externally anchored. |

## Scope and non-goals

CAPS standardizes the boundary between CoCo attestation/resource services and
an ACS-compatible agent-governance runtime. It is independent of Rego, Cedar,
or a custom ACS policy dispatcher.

CAPS does not:

* replace the ACS manifest or policy engine;
* add intervention points to the ACS runtime;
* prove that probabilistic classifiers or model outputs are correct;
* make a remote tool trustworthy merely because its caller is attested;
* eliminate TEE side channels, denial of service, or traffic analysis;
* allow the Kubernetes control plane to become a source of trusted snapshot
  claims after attestation.

## Conformance roles

A conforming deployment has these roles:

* **ACS runtime**: the pinned, stateless evaluator defined by ACS.
* **APE**: the measured ACS host, policy enforcement point, state manager, and
  receipt signer inside the confidential guest.
* **Agent adapter**: an untrusted framework integration that proposes lifecycle
  events to the APE. It does not enforce final verdicts.
* **Attestation Service (AS)**: appraises TEE evidence and reference values.
* **Trustee/KBS**: relies on attestation results and issues resources or
  capabilities after independent authorization.
* **Policy issuer**: signs the CAPS envelope and supplies ACS policies.
* **Tool or capability consumer**: verifies action-bound authority before
  accepting an invocation.
* **Receipt verifier**: validates the APE's session key, receipt chain, and
  external checkpoints.

A conforming CAPS APE MUST:

1. be measured and isolated from the agent workload;
2. start before the workload and own the only permitted side-effect paths;
3. verify and resolve the CAPS envelope before constructing ACS;
4. call ACS in `enforce` mode at every configured mandatory intervention point;
5. construct trusted snapshot and annotation fields independently of agent
   values;
6. enforce the ACS verdict and transform at the actual I/O boundary;
7. maintain state outside the stateless ACS runtime;
8. authorize resource acquisition against the exact action and verdict;
9. emit a signed receipt for each evaluation and externally visible action;
10. fail closed when any required evidence, state, dispatcher, or sink is
    unavailable or ambiguous.

## CAPS signed envelope

### Media type

The resolved policy bundle uses:

```text
application/vnd.coco.caps-acs-envelope+json;version=0.1
```

The envelope is an I-JSON-compatible JSON object serialized with the JSON
Canonicalization Scheme before hashing or signing.

### Structure

```json
{
  "caps_profile_version": "0.1",
  "profile": "coco-confidential-agent",
  "policy_id": "urn:example:policy:invoice-reconciler",
  "epoch": 42,
  "issued_at": "2026-06-06T00:00:00Z",
  "not_before": "2026-06-06T00:00:00Z",
  "expires_at": "2026-09-04T00:00:00Z",
  "issuer": "spiffe://security.example/policy-authority",
  "acs": {
    "version": "0.3.1-beta",
    "schema_digest": "sha-256:...",
    "manifest_digest": "sha-256:...",
    "manifest": {},
    "artifacts": {
      "policy/invoice.tar.gz": "sha-256:..."
    }
  },
  "attested_subjects": [],
  "trusted_snapshot": {},
  "state": {},
  "capabilities": {},
  "receipts": {},
  "signature": {}
}
```

### Field reference

The envelope is a signed deployment contract, not an ACS evaluation request.
It identifies the exact governance program, the confidential workload allowed
to run it, the trusted state the APE must maintain, and the evidence required
before side effects or resource release.

#### Envelope identity and validity fields

| Field | Requirement | Detailed semantics |
| --- | --- | --- |
| `caps_profile_version` | REQUIRED | Version of this CAPS envelope/profile schema. It versions the confidential extension, not ACS. An unsupported value fails closed. |
| `profile` | REQUIRED | Profile identifier. CAPS 0.1 requires `coco-confidential-agent`; it prevents a verifier from confusing this envelope with another signed object type. |
| `policy_id` | REQUIRED | Stable URI naming one policy lineage across updates. Receipts, approvals, delegations, and revocation rules refer to this value. It is an identifier, not a content hash. |
| `epoch` | REQUIRED | Unsigned, monotonically increasing revision for `policy_id`. The APE and Trustee reject an epoch lower than the highest accepted epoch, preventing rollback to an older but correctly signed policy. |
| `issued_at` | REQUIRED | RFC 3339 time at which the issuer created this envelope. It supports audit and maximum-age policy but does not by itself authorize use. |
| `not_before` | REQUIRED | Earliest time or verifier lease at which the envelope may be activated. The APE must not start a governed session before it. |
| `expires_at` | REQUIRED | Hard upper bound for starting or continuing a governed session. A capability, approval, or session derived from this envelope cannot outlive it. |
| `issuer` | REQUIRED | Stable identity of the tenant authority permitted to sign this policy lineage, for example a SPIFFE ID. Trust in the issuer comes from Init-Data or Trustee configuration, not this self-asserted string. |

`issued_at`, `not_before`, and `expires_at` require a trusted-time strategy. If
the TEE lacks trustworthy wall-clock time, the verifier SHOULD issue a bounded
lease after checking these values. Loss or expiry of that lease fails closed for
new side effects.

#### `acs`: pinned Agent Control program

| Field | Requirement | Detailed semantics |
| --- | --- | --- |
| `acs.version` | REQUIRED | Exact ACS semantic version implemented by the runtime, such as `0.3.1-beta`. It must equal the manifest's `agent_control_specification_version`. |
| `acs.schema_digest` | REQUIRED | Digest of the exact machine-readable ACS manifest schema used at build time and accepted by the APE. This protects against schema drift under an unchanged version label. |
| `acs.manifest` | REQUIRED | Fully resolved, schema-valid ACS manifest used to construct the runtime. It contains intervention points, policy bindings, tools, annotators, and approval configuration, but no unresolved inheritance. |
| `acs.manifest_digest` | REQUIRED | Digest of the canonical JSON representation of `acs.manifest`. The APE recomputes it before use; Trustee and receipts use it as the policy-program identity. |
| `acs.artifacts` | REQUIRED | Map from normalized bundle-relative artifact path to content digest. It covers code/configuration referenced by the manifest but not embedded in it, such as Rego bundles, Cedar files, custom dispatchers, and annotator configurations. |

`acs.manifest_digest` identifies the declarative ACS wiring; it does not replace
artifact digests. For example, a manifest may continue to say
`bundle: ./policy` while the Rego content at that path changes. The artifact map
closes that gap. Implementations SHOULD also derive one deterministic
`acs_artifact_set_digest` from the sorted path/digest map for attestation,
capability requests, and receipts.

#### `attested_subjects`: who may run the policy

`attested_subjects` is a nonempty array of acceptable workload configurations.
An entry normally contains:

| Field | Requirement | Detailed semantics |
| --- | --- | --- |
| `id` | REQUIRED | Logical workload identity to issue after successful appraisal. It is matched with tenant, namespace, service account, or equivalent authenticated deployment identity. |
| `tee_profiles` | REQUIRED | Allowed evidence/appraisal profiles, for example `coco-sev-snp` or `coco-tdx`. Matching a hardware family alone is insufficient; its profile includes required claims and verifier policy. |
| `measurements.guest_image` | REQUIRED | Digest/reference value for the confidential guest image and base guest services. |
| `measurements.ape` | REQUIRED | Measurement of the APE that hosts ACS and enforces final I/O. |
| `measurements.acs_runtime` | REQUIRED | Measurement of the ACS runtime implementation compatible with `acs.version`. |
| `measurements.acs_policy_artifacts` | REQUIRED | Digest of the materialized policy artifact set referenced by `acs.artifacts`. |
| `measurements.agent_image` | REQUIRED | Digest of the unprivileged agent workload image. |
| `measurements.model` | OPTIONAL | Allowed model artifact or attested remote model identities. Empty means the envelope must define an explicit external-model policy rather than trusting any model. |
| `measurements.adapters` | OPTIONAL | Digests of adapters, LoRAs, plugins, or framework extensions that affect agent behavior. |
| `measurements.system_prompt` | OPTIONAL | Digest of immutable system instructions when they are part of the governed identity. Dynamic user content is not measured here. |
| `measurements.init_data` | REQUIRED | Digest of measured bootstrap configuration, including issuer roots and the envelope reference. |

Entries are alternatives, while all populated fields within one entry are
conjunctive. The APE selects exactly one matching entry and records it in the
session and receipts. Values supplied by the agent never satisfy this match.

#### `trusted_snapshot`: where ACS claims come from

This object defines how the APE constructs the trusted portion of every ACS
snapshot. It is policy about claim provenance, not the snapshot values
for one particular request. A deployment uses it to declare:

* required claim paths under `snapshot.caps`;
* the source for each path, such as an attestation result, authenticated
  principal, APE session state, gateway observation, or trusted-time lease;
* freshness and maximum-age requirements;
* annotator identity, trust class, timeout, response-size, and failure behavior;
* which fields the untrusted adapter may propose under `snapshot.agent`.

The APE MUST reject an event that attempts to write, alias, or shadow a trusted
path. `trusted_snapshot` MUST NOT contain secrets; it may identify the trusted
service from which a claim is obtained.

#### `state`: controls around stateless ACS

ACS computes one verdict at a time and retains no mutable state. `state` tells
the APE which cross-evaluation state it must protect and how. It includes:

* budget dimensions and limits;
* event sequence and replay windows;
* approval-use and capability-use counters;
* delegation depth, fan-out, expiry, and budget attenuation;
* idempotency rules for side effects;
* persistence, lease, or monotonic-counter requirements;
* behavior when trusted time or the external state service is unavailable.

This field contains limits and mechanisms, not the current counter values. The
current values appear in `snapshot.caps.state` and are committed by the APE.

#### `capabilities`: how an allowed verdict gains authority

This object maps governed operations to stable Trustee/KBS resources and
capability constraints. A mapping can specify:

* ACS intervention point, tool, action class, purpose, and destination;
* stable `kbs:///` resource identifier;
* required attestation and subject claims;
* capability audience, proof-of-possession key, maximum uses, and lifetime;
* which ACS input/verdict/event digests must be bound into the capability;
* whether the APE must consume the capability internally or may return a
  restricted handle to a sandboxed local tool.

The resource identifier and audience come from this signed object, never from a
prompt. An ACS `allow` merely satisfies one condition; KBS independently checks
this mapping and the attestation result before releasing authority.

#### `receipts`: evidence and privacy policy

This object defines what the APE records and where it anchors receipt-chain
roots. It includes:

* required receipt fields and commitment algorithms;
* external anchor identity and cadence;
* maximum unanchored entries or elapsed time;
* retention period and authorized receipt readers;
* whether selected content may be retained, encrypted, or must be omitted;
* behavior when the receipt sink is unavailable.

The default is metadata and keyed commitments, not raw prompts, credentials,
chain of thought, or tool results. Reaching the configured unanchored limit MUST
deny new externally visible side effects.

#### `signature`: issuer authorization

`signature` identifies the signature format, signer key/certificate reference,
and signature bytes. COSE Sign1 and DSSE are RECOMMENDED envelope formats. The
signature covers the JCS-canonical envelope with `signature` omitted. It
protects every ACS and CAPS field as one unit.

Signature validity is necessary but insufficient. The APE also verifies that:

1. the signer is authorized for `policy_id`;
2. the validity window and epoch are acceptable;
3. all embedded and referenced digests recompute correctly;
4. one `attested_subjects` entry matches;
5. the ACS manifest validates against the pinned schema.

### Resolved ACS manifest

A **source ACS manifest** is the author-friendly YAML or JSON written by policy
authors. It may use ACS `extends` to inherit reusable organization policy. A
**resolved ACS manifest** is the single, inheritance-resolved manifest
produced after loading all parents and merging them according to ACS rules. It
is self-contained with respect to manifest composition and is the exact
manifest passed to the ACS runtime. Referenced policy artifacts remain separate
but are content-pinned by `acs.artifacts`.

A resolved manifest is not compiled Rego/Cedar bytecode, an ACS policy input, a
runtime snapshot, or an execution result. It is the materialized ACS
configuration: intervention points, policy definitions and bindings, tool
catalog, annotator declarations, approval resolvers, and metadata.

For example, an organization may publish this parent:

```yaml
agent_control_specification_version: "0.3.1-beta"
policies:
  organization_policy:
    type: rego
    bundle: ./policy/organization
    query: data.organization.verdict
intervention_points:
  output:
    policy_target: $snap.final_output
    policy_target_kind: final_output
    policy:
      id: organization_policy
```

The workload source manifest can inherit it:

```yaml
agent_control_specification_version: "0.3.1-beta"
extends:
  - url: https://policy.example/acs/base.yaml
    sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
policies:
  invoice_policy:
    type: rego
    bundle: ./policy/invoice
    query: data.invoice.verdict
intervention_points:
  pre_tool_call:
    policy_target: $snap.tool_call.args
    policy_target_kind: tool_args
    tool_name_from: $snap.tool_call.name
    policy:
      id: invoice_policy
tools:
  invoice.read:
    type: Tool
    clearance: restricted-pii
```

Before signing or booting the guest, trusted build tooling fetches and verifies
the pinned parent, checks versions and cycles, applies the ACS additive merge,
and emits the resolved manifest:

```yaml
agent_control_specification_version: "0.3.1-beta"
metadata:
  caps_resolution:
    source_digest: sha-256:...
    resolver: acs-resolver-1.0
policies:
  organization_policy:
    type: rego
    bundle: ./policy/organization
    query: data.organization.verdict
  invoice_policy:
    type: rego
    bundle: ./policy/invoice
    query: data.invoice.verdict
intervention_points:
  output:
    policy_target: $snap.final_output
    policy_target_kind: final_output
    policy:
      id: organization_policy
  pre_tool_call:
    policy_target: $snap.tool_call.args
    policy_target_kind: tool_args
    tool_name_from: $snap.tool_call.name
    policy:
      id: invoice_policy
tools:
  invoice.read:
    type: Tool
    clearance: restricted-pii
```

Notice that `extends` is absent. The child has not overridden the parent's
`output` policy; both non-conflicting definitions are present. Under ACS,
conflicting duplicate definitions, version differences, cycles, missing files,
path traversal, failed fetches, and integrity mismatches fail resolution.

The embedded ACS manifest MUST be resolved before attestation and MUST validate
against the pinned ACS schema. This provides four security properties:

1. **No policy network access at runtime.** The APE never fetches a parent while
   deciding whether to release a secret or call a tool.
2. **One reviewable policy program.** Reviewers and verifiers hash the exact
   merged manifest rather than reconstructing inheritance independently.
3. **Stable attestation binding.** The manifest digest can be placed in
   Init-Data, attestation results, capabilities, approvals, and receipts.
4. **Deterministic construction.** Every conforming resolver must either produce
   the same merged manifest or fail closed.

The resolution and packaging pipeline is:

```text
source manifest + pinned parents + policy artifacts
            │
            ▼
resolve ACS extends and reject conflicts/cycles/path traversal
            │
            ▼
validate merged manifest against pinned ACS schema
            │
            ▼
canonicalize manifest and compute manifest_digest
            │
            ▼
hash referenced policy/dispatcher/annotator artifacts
            │
            ▼
construct and sign CAPS envelope
            │
            ▼
bind envelope/manifest/artifact digests into Init-Data and RVPS
```

`acs.manifest_digest` covers the canonical resolved manifest.
`acs.schema_digest` covers the exact schema used to validate it. `acs.artifacts`
MUST list the digest of every Rego bundle, Cedar policy/entity/schema file,
custom dispatcher module, and annotator configuration referenced by the
manifest. Artifact paths resolve only inside the measured policy bundle and
MUST NOT traverse outside it. The APE binary or measured configuration MUST
identify the matching ACS runtime build.

CAPS prohibits these ACS configurations in an enforcing confidential profile:

* `evaluate_only` for mandatory intervention points;
* unpinned or runtime-resolved `extends`;
* approval timeout behavior of `allow`;
* a production policy of ACS type `test`;
* unknown tools at a tool intervention point;
* a dispatcher or annotator with ambient host credentials;
* a transform that is applied outside the measured APE;
* a policy default that permits an error or missing trusted field.

## Attested subject binding

An attested subject binds policy use to verified claims:

```json
{
  "id": "spiffe://agents.example/ns/finance/sa/reconciler",
  "tee_profiles": ["coco-sev-snp", "coco-tdx"],
  "measurements": {
    "guest_image": "sha-256:...",
    "ape": "sha-256:...",
    "acs_runtime": "sha-256:...",
    "acs_policy_artifacts": "sha-256:...",
    "agent_image": "sha-256:...",
    "model": ["sha-256:..."],
    "adapters": [],
    "system_prompt": "sha-256:...",
    "init_data": "sha-256:..."
  }
}
```

Every nonempty constraint is conjunctive. Subject matching uses signed
attestation results or trusted claims derived from them. Agent-provided values
MUST NOT satisfy subject matching. The attestation result MUST bind a fresh
challenge, the APE session public key, CAPS envelope digest, and ACS manifest
digest.

A model, adapter, system prompt, ACS runtime, policy dispatcher, annotator, APE,
or resolved manifest change creates a different attested subject and requires a
new appraisal and session.

## Trusted ACS snapshot profile

### Trust boundary

ACS treats the host-provided snapshot as its complete view of the agent and
environment. Under CAPS, the APE is that host. The agent adapter may submit
candidate data, but the APE MUST reconstruct security-sensitive fields from
attested identity, gateway observations, authenticated principals, and its own
rollback-resistant state.

The snapshot uses two top-level namespaces:

```json
{
  "caps": {
    "trusted": {},
    "state": {},
    "event": {}
  },
  "agent": {
    "proposed": {}
  }
}
```

`caps.trusted`, `caps.state`, and APE-generated members of `caps.event` are
trusted. `agent.proposed` and all model, prompt, retrieval, and tool-result
content are untrusted unless an authenticated source is explicitly promoted by
policy. The APE MUST reject an adapter request that attempts to populate or
shadow a `caps` field.

### Required trusted claims

Every ACS evaluation snapshot MUST include:

```json
{
  "caps": {
    "trusted": {
      "profile_version": "0.1",
      "policy_id": "urn:example:policy:invoice-reconciler",
      "policy_epoch": 42,
      "envelope_digest": "sha-256:...",
      "acs_manifest_digest": "sha-256:...",
      "subject_id": "spiffe://agents.example/ns/finance/sa/reconciler",
      "session_id": "urn:uuid:...",
      "session_key_thumbprint": "sha-256:...",
      "attestation_result_digest": "sha-256:...",
      "attestation_issued_at": "2026-06-06T00:00:00Z",
      "principal": {},
      "trusted_time": {}
    },
    "state": {
      "sequence": 17,
      "budget_remaining": {},
      "delegation": {},
      "approval_refs": [],
      "receipt_head": "sha-256:..."
    },
    "event": {
      "event_id": "019751d0-61d4-7dc2-8ea5-f1e8d55bb708",
      "parent_event_id": null,
      "idempotency_key": null
    }
  }
}
```

The APE generates `event_id`, `sequence`, and idempotency keys. An event ID is
unique within the trust domain and MUST NOT be authorized twice. UUIDv7 is
RECOMMENDED. The APE MUST preserve a tool invocation's event ID across
`pre_tool_call` and `post_tool_call`; this strengthens the optional ACS
`tool_call.id` convention into a CAPS requirement.

### ACS intervention requirements

CAPS uses the existing ACS intervention points without renaming them:

| ACS intervention point | CAPS requirement |
| --- | --- |
| `agent_startup` | Verify subject, policy epoch, session lease, configured model, tools, and mandatory controls before opening egress. |
| `input` | Authenticate the principal, assign provenance/labels, and reject attempts to supply trusted fields. |
| `pre_model_call` | Enforce model identity, endpoint attestation, input labels, token/cost budget, and retention policy. |
| `post_model_call` | Treat output as untrusted; apply configured annotations/transforms before parsing actions. |
| `pre_tool_call` | Canonicalize arguments, reserve budget, resolve approval, and obtain action-bound capability before dispatch. |
| `post_tool_call` | Record observed outcome, settle budget, label/provenance the result, and block embedded authority. |
| `output` | Enforce destination, declassification, redaction transform, size, and disclosure policy. |
| `agent_shutdown` | Close delegation, anchor receipts, destroy session authority, and report unresolved actions. |

A manifest MAY omit a point only when the signed CAPS envelope declares it
non-mandatory and the APE proves there is no corresponding path. At minimum,
`agent_startup`, `pre_tool_call`, `output`, and `agent_shutdown` are mandatory.
A deployment that invokes an external model MUST also configure
`pre_model_call` and `post_model_call`. A deployment that accepts external input
MUST configure `input`.

### Annotations and evidence

ACS annotators are host dispatchers, not intrinsically trusted facts. CAPS
classifies every annotator in `trusted_snapshot.annotations` as one of:

* `measured_local`: deterministic code measured with the APE;
* `attested_remote`: response authenticated to an approved attested service;
* `authenticated_remote`: response authenticated to a named service but not
  confidentially attested;
* `probabilistic`: classifier or LLM judge whose output is advisory evidence;
* `untrusted`: content that policy may inspect but MUST NOT use to grant
  authority.

Each annotation placed in ACS `annotations` MUST be accompanied in the snapshot
by provenance, producer identity, request/response digest, freshness, and trust
class. A probabilistic or untrusted annotation MAY cause denial or escalation,
but MUST NOT by itself authorize secret release, declassification, delegation,
or a privileged side effect.

Annotator and policy dispatchers MUST have explicit network destinations,
timeouts, response-size limits, and credentials. Their credentials are held by
the APE or gateway and MUST NOT be inherited from host environment variables.
Failure or timeout yields the ACS fail-closed denial.

## Verdict enforcement profile

The APE acts on ACS verdicts at the actual model, tool, storage, message, and
output boundary:

| ACS verdict | CAPS enforcement |
| --- | --- |
| `allow` | Continue only after all CAPS state and capability preconditions commit. |
| `warn` | Record the warning; continue only if the envelope explicitly permits warnings for this point. |
| `deny` | Do not perform the operation and emit a denial receipt. |
| `escalate` | Suspend the operation, obtain signed approval bound to the event digest, then re-evaluate or deny. |
| `transform` | Apply the ACS-validated transform inside the APE, recompute the event digest, and enforce the transformed value. |

An adapter acknowledgement is not enforcement. For example, after an allowed
`pre_tool_call`, the APE-owned gateway sends the network request itself. The APE
MUST NOT return a long-lived credential to the agent and trust it to call the
same arguments.

A `warn` verdict is deny-by-default in CAPS unless the signed envelope names the
intervention point and permitted warning reasons. An `escalate` timeout MUST
deny or remain suspended; `approval.on_timeout: allow` is invalid. A transformed
value is a new authoritative policy target and MUST be the value dispatched.

## Stateful controls around stateless ACS

ACS is intentionally stateless. The APE owns the state that appears in each
snapshot and commits it transactionally around verdict enforcement.

### Budgets

The envelope defines non-negative integer limits for dimensions such as model
tokens, tool calls, write actions, money, wall time, delegation depth, and
fan-out. The APE reserves the maximum authorized amount before dispatch and
settles observed usage afterward. A crash cannot restore consumed budget.
Distributed operation requires a linearizable lease or conservatively divided
local sub-budgets.

The ACS policy reads current and requested budget through the snapshot. ACS
returns the policy verdict; the APE performs the atomic reservation and denies
if state changed between evaluation and commit.

### Approval

CAPS uses ACS `escalate` and the ACS approval manifest section. The approval
resolver runs outside the stateless runtime but inside the APE's controlled
dispatch boundary.

A valid approval signs:

* event and canonical policy-target digests;
* CAPS envelope and ACS manifest digests;
* approver identity and policy-authorized role;
* permitted transform or limits;
* issue time, expiry, and unique approval ID.

Material changes to the target, tool, destination, labels, budget, or policy
invalidate approval. Approval never overrides an ACS `deny` or increases budget.

### Delegation

ACS has no built-in delegation state. CAPS represents a child agent's active
delegation under `caps.state.delegation`, and ACS policy evaluates it at
`agent_startup`, `pre_tool_call`, `pre_model_call`, and `output` as applicable.

The APE computes child authority as the intersection of parent remaining
authority, signed policy, requested tools, data-label ceiling, budget, purpose,
expiry, depth, and fan-out. Every dimension monotonically decreases. The child
receives a new attested session key and identity. A child cannot delegate unless
its delegation explicitly permits it.

### Information flow

ACS defines a stateless label-flow model and makes the host responsible for
tracking provenance and labels between calls. CAPS requires the APE to perform
that host function. The APE maintains labels for input, retrieval, model output,
tool result, memory, and transformed data, and supplies them in the ACS snapshot
at each sink.

The effective label is the join of all inputs. A model may raise a label but
cannot lower one. Declassification requires an ACS policy verdict permitting a
specific transform plus a CAPS rule naming source/target labels, exact fields,
measured transformer, destination, purpose, and receipt requirements. A prompt
instruction to remove sensitive data is not declassification.

### Anti-rollback

Policy epoch, budget, delegation, approval consumption, capability use, and
receipt sequence MUST be protected by a monotonic counter, tenant service, or
bounded online lease. When freshness cannot be established, the APE denies new
side effects. Restored checkpoints receive a new session and cannot reuse old
capabilities.

## KBS resources and action capabilities

An allowed ACS verdict does not itself release a secret. Before an external side
effect, the APE requests a resource or capability from Trustee using:

```json
{
  "resource_id": "kbs:///finance/invoice-api/signing-operation",
  "subject_id": "spiffe://agents.example/ns/finance/sa/reconciler",
  "session_key_thumbprint": "sha-256:...",
  "event_digest": "sha-256:...",
  "acs_input_digest": "sha-256:...",
  "acs_verdict_digest": "sha-256:...",
  "caps_envelope_digest": "sha-256:...",
  "acs_manifest_digest": "sha-256:...",
  "acs_artifact_set_digest": "sha-256:...",
  "tool_audience": "spiffe://tools.example/invoice-api",
  "max_uses": 1
}
```

KBS policy independently checks attestation result, subject, policy epoch,
resource, event, verdict, audience, expiry, and session-key binding. Stable
resource IDs come from signed configuration, never model text.

The returned authority SHOULD be a non-exportable cryptographic operation,
single-use proof-of-possession token, or short-lived mTLS credential. It MUST be
bound to the event digest, tool audience, session key, policy digests, expiry,
and unique capability ID. The gateway consumes it internally. The tool rejects
request mismatch and replay.

## Receipt profile

CAPS receipts extend an ACS evaluation record with confidential-computing and
enforcement evidence:

```json
{
  "caps_profile_version": "0.1",
  "receipt_id": "urn:uuid:...",
  "session_id": "urn:uuid:...",
  "sequence": 17,
  "previous_receipt": "sha-256:...",
  "event_id": "019751d0-61d4-7dc2-8ea5-f1e8d55bb708",
  "intervention_point": "pre_tool_call",
  "acs_input_digest": "sha-256:...",
  "acs_verdict": "allow",
  "acs_verdict_digest": "sha-256:...",
  "policy_reason": "invoice_read_allowed",
  "caps_envelope_digest": "sha-256:...",
  "acs_manifest_digest": "sha-256:...",
  "acs_artifact_set_digest": "sha-256:...",
  "attestation_result_digest": "sha-256:...",
  "approval_ids": [],
  "capability_id": "urn:uuid:...",
  "budget_delta": {"tool_calls": 1},
  "outcome": "success",
  "result_commitment": "hmac-sha-256:...",
  "timestamp": "2026-06-06T00:00:01Z",
  "signature": {}
}
```

`outcome` is `denied`, `cancelled`, `success`, `failure`, or `outcome_unknown`.
The APE signs with the attestation-bound session key. Receipts form a hash chain
and roots are periodically anchored to a tenant-controlled service.

Receipts contain commitments and metadata by default, not prompts, chain of
thought, credentials, or tool results. A plain digest of low-entropy private
data can leak through guessing, so the APE SHOULD use a tenant-keyed commitment
where appropriate.

## Example ACS manifest under CAPS

The ACS manifest remains valid ACS. Confidential requirements live in the
outer envelope and trusted snapshot:

```yaml
agent_control_specification_version: "0.3.1-beta"
metadata:
  name: invoice-reconciler
policies:
  invoice_policy:
    type: rego
    bundle: ./policy
    query: data.invoice_agent.verdict
intervention_points:
  agent_startup:
    policy_target: "$.agent.configuration"
    policy_target_kind: agent_configuration
    policy:
      id: invoice_policy
  pre_model_call:
    policy_target: "$.model_request"
    policy_target_kind: model_request
    policy:
      id: invoice_policy
  pre_tool_call:
    policy_target: "$.tool_call.args"
    policy_target_kind: tool_args
    tool_name_from: "$.tool_call.name"
    policy:
      id: invoice_policy
  post_tool_call:
    policy_target: "$.tool_result"
    policy_target_kind: tool_result
    tool_name_from: "$.tool_call.name"
    policy:
      id: invoice_policy
  output:
    policy_target: "$.final_output"
    policy_target_kind: final_output
    policy:
      id: invoice_policy
  agent_shutdown:
    policy_target: "$.agent.session"
    policy_target_kind: agent_session
    policy:
      id: invoice_policy
tools:
  invoice.read:
    type: Tool
    id: invoice.read
    clearance: restricted-pii
    security_labels: [internal-api]
approval:
  default_resolver: finance-approval
  timeout_seconds: 300
  on_timeout: deny
  resolvers:
    finance-approval:
      type: custom
```

At `pre_tool_call`, the APE adds attested identity, policy digests, labels,
budget, event identity, and approvals under `snapshot.caps`. Rego or Cedar can
then reason over both the ordinary ACS projection and trusted confidential
claims. The APE—not Rego and not the agent adapter—reserves budget, acquires the
KBS capability, executes the call, and writes the receipt.

## Minimum profiles

### CAPS-ACS-Core

* schema-valid, version-pinned, fully resolved ACS manifest;
* ACS `enforce` mode and fail-closed verdict enforcement;
* trusted/untrusted snapshot namespace separation;
* APE-owned event identity, state, gateway, and receipt chain;
* mandatory startup, side-effect, output, and shutdown interception.

### CAPS-ACS-Confidential

Includes CAPS-ACS-Core plus:

* signed CAPS envelope and policy artifacts bound to measured Init-Data;
* fresh CoCo attestation bound to the APE session key;
* measured APE, ACS runtime, manifest, agent, model, and system configuration;
* KBS authorization against subject, policy epoch, event, and verdict;
* encrypted transport/state and anti-rollback lease or counter;
* no user-declared Kubernetes sidecar as the enforcement boundary.

### CAPS-ACS-High-Assurance

Includes CAPS-ACS-Confidential plus:

* dedicated confidential VM or documented equivalent isolation;
* proof-of-possession, single-use capabilities;
* external receipt anchoring and online revocation;
* multi-party approval for irreversible or privileged actions;
* attested remote tools/models or explicit trust-boundary declarations;
* composed accelerator attestation when protected data reaches an accelerator.

An implementation MUST NOT advertise a profile when any required control is
disabled.

## Conformance tests

A CAPS conformance suite MUST include ACS conformance vectors for the pinned
runtime plus confidential-profile vectors for:

1. unsupported ACS version or schema digest;
2. unresolved, remote, unpinned, cyclic, or conflicting `extends`;
3. invalid envelope signature, expiry, or epoch rollback;
4. APE, ACS runtime, agent, model, or manifest measurement mismatch;
5. agent attempt to inject or shadow `snapshot.caps`;
6. missing mandatory intervention point or use of `evaluate_only`;
7. `warn` without explicit permission;
8. `escalate` timeout configured to allow;
9. annotator provenance downgrade, failure, or stale evidence;
10. transformed target not used for actual dispatch;
11. event replay or mismatch between pre/post tool evaluations;
12. concurrent budget spend or rollback;
13. approval target or policy mismatch;
14. delegation amplification, depth, fan-out, or expiry violation;
15. label downgrade without authorized transform/declassification;
16. KBS release without matching attestation, event, and ACS verdict;
17. capability audience, request, session-key mismatch, or replay;
18. agent bypass of the APE-owned model, tool, storage, or output boundary;
19. crash after non-idempotent dispatch and `outcome_unknown` handling;
20. broken receipt chain, invalid APE signature, or anchoring-limit exhaustion.

## Security considerations

* ACS portability does not make the host trustworthy. CAPS protects the host
  boundary with measurement, isolation, and attestation.
* Attestation proves that expected software started; it does not prove that its
  policy is safe. Tenant-controlled issuer roots and review remain necessary.
* A signed ACS manifest can still authorize dangerous behavior. KBS and remote
  tools independently enforce resource and request constraints.
* Probabilistic annotators may detect risk but cannot be the sole source of
  positive authority.
* Typed schemas constrain syntax, not business meaning. Narrow APIs, simulation,
  budgets, and approval remain necessary for high-impact actions.
* Rollback, trusted time, and crash consistency require an online tenant service
  or hardware support; sealing alone is insufficient.
* A compromised agent may attempt denial of service against the APE. Evaluator,
  dispatcher, snapshot, annotation, and receipt sizes MUST be bounded.
* TEE and accelerator side channels remain deployment-specific residual risks.

## Evolution and upstream alignment

ACS `0.3.1-beta` is a draft. CAPS should be maintained as a profile with a small
compatibility matrix rather than copying ACS text or implementation. When ACS
adds native extension metadata, trusted snapshot claims, signed manifests, or
receipt bindings, CAPS SHOULD adopt those primitives and retire overlapping
profile fields.

The following capabilities are candidates for upstream ACS discussion:

* a standard convention for trusted versus untrusted snapshot namespaces;
* stable invocation identity across pre/post intervention points;
* signed and digest-pinned resolved manifests;
* provenance and trust classes for annotations;
* normalized verdict and transformed-target digests for audit;
* host conformance claims describing enforcement and isolation strength.

CAPS-specific TEE measurements, CoCo attestation results, KBS resource binding,
anti-rollback mechanisms, and capability issuance remain confidential-computing
profile concerns rather than requirements for every ACS implementation.
