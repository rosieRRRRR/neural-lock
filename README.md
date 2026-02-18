# Neural Lock -- Operator State Evidence

**Reference Implementation: Bitcoin Custody Systems**

* **Specification Version:** 1.1.0
* **Status:** Implementation Ready
* **Date:** 2026
* **Author:** rosiea
* **Contact:** [PQRosie@proton.me](mailto:PQRosie@proton.me)
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea
* **PQ Ecosystem:** CORE — The PQ Ecosystem is a post-quantum security framework built on deterministic enforcement, fail-closed semantics, and refusal-driven authority. Bitcoin is the reference deployment. It is not the scope.

---

## Summary

Neural Lock defines how systems record and report operator state signals relevant to security and coercion risk.

It specifies how posture indicators and optional duress signals are emitted as constrained, non-authoritative signals without directly triggering actions.

Neural Lock does not enforce policy. Its outputs are evaluated by PQSEC as part of broader security decisions.

---

## Index

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
   * 2A. [Explicit Dependencies](#2a-explicit-dependencies)
3. [Architecture](#3-architecture)
4. [Signal Sources and State Classification](#4-signal-sources-and-state-classification)
   * 4.11 [UNAVAILABLE Classification Semantics (Normative)](#411-unavailable-classification-semantics-normative)
   * 4.12 [Manual Duress Signal (Normative if Implemented)](#412-manual-duress-signal-normative-if-implemented)
5. [Policy Integration](#5-policy-integration)
6. [Bitcoin Integration Patterns](#6-bitcoin-integration-patterns)
   * 6A. [Beyond Bitcoin: General Application Domains](#6a-beyond-bitcoin-general-application-domains) (Informative)
7. [Security Properties](#7-security-properties)
8. [Privacy Protections](#8-privacy-protections)
9. [Threat Model](#9-threat-model)
10. [Integration with PQ Ecosystem](#10-integration-with-pq-ecosystem)
    * 10.1 [Generalised Security Gating](#101-generalised-security-gating) (General Applicability)
11. [Implementation Requirements and Guidance](#11-implementation-requirements-and-guidance)
12. [Comparison to Existing Solutions](#12-comparison-to-existing-solutions)
13. [Future Work](#13-future-work)
14. [References](#14-references)
15. [Annexes](#15-annexes)
    * [Annex A: State Classification Pseudocode](#annex-a-state-classification-pseudocode-normative) (Normative)
    * [Annex B: Spoofing and Liveness Detection Pseudocode](#annex-b-spoofing-and-liveness-detection-pseudocode-reference) (Reference)
    * [Annex C: NeuralAttestation Construction and Signing](#annex-c-neuralattestation-construction-and-signing-reference) (Reference)
    * [Annex D: PSBT Integration and Proprietary Field Versioning](#annex-d-psbt-integration-and-proprietary-field-versioning-reference) (Reference)
    * [Annex E: Conformance Test Scenarios](#annex-e-conformance-test-scenarios-normative) (Normative)
    * [Annex F: Decoy Wallet Derivation and Constraints](#annex-f-decoy-wallet-derivation-and-constraints-reference) (Reference)
    * [Annex G: Training Mode and Baseline Computation](#annex-g-training-mode-and-baseline-computation-informative) (Informative)
    * [Annex H: Break Glass Flow](#annex-h-break-glass-flow-informative) (Informative)
    * [Annex I: Generic Operator State Gating Interface](#annex-i-generic-operator-state-gating-interface-informative) (Informative)
    * [Annex J: Quick Start Guide](#annex-j-quick-start-guide-informative) (Informative)
    * [Annex K: Compliance Checklist](#annex-k-compliance-checklist-informative) (Informative)
16. [Acknowledgements](#16-acknowledgements)


---

## 1. Introduction

### 1.1 Problem Statement

Current Bitcoin security models assume that possession of cryptographic keys equals legitimate authority to spend funds. This model fails under coercion:

* Physical coercion: attacker threatens user with violence ("$5 wrench attack")
* Social engineering under duress
* Impairment: user makes decisions while cognitively compromised
* Panic transactions: stress-induced irreversible financial decisions

Existing countermeasures (duress PINs, time delays, multisig) can be circumvented with patience, repeated attempts, or by demanding all authentication factors.

### 1.2 Current Security Model

```
keys = authority
```

Problem: attackers can extract keys through coercion.

### 1.3 Neural Lock Security Model

```
keys + operator_state_ok = authority
```

Neural Lock does not verify intent. It classifies risk-correlated operator state and provides evidence for policy enforcement.

### 1.4 Design Principles

1. Non-authoritative: human-state signals never sign transactions or override cryptographic keys
2. Fail-safe: uncertain or degraded signals reduce permissions, never expand them
3. Local and private: all processing occurs on-device; raw biometric data is never exported
4. Policy-bound: signals map to coarse states that feed existing policy engines
5. User-configurable: deployments MAY allow advisory-only mode by policy; disabling Neural Lock is a policy change and MUST NOT expand authority
6. Transparent: state transitions are logged locally for user audit

### 1.5 Authority Boundary

Neural Lock grants no authority.

Neural Lock:

* does not authorize
* does not sign Bitcoin transactions
* does not override cryptographic custody
* does not execute actions
* does not perform enforcement

Neural Lock produces descriptive, non-authoritative attestation artefacts only.

All enforcement, refusal, escalation, lockout, and backoff semantics are defined exclusively by PQSEC.

Any implementation deriving authority directly from Neural Lock artefacts is non conformant.

### 1.6 Versioning and Compatibility

Neural Lock is versioned to prevent silent wire-format drift and ambiguous interpretation across implementations.

Requirements:

1. NeuralAttestation MUST include an attestation_version field.
2. This specification defines attestation_version = 1.
3. Implementations MUST reject NeuralAttestations whose attestation_version they do not support.
4. Any backwards-incompatible change to canonical fields, canonical encoding rules, or signing message definition MUST increment attestation_version.
5. Backwards-compatible changes MUST NOT break canonical encoding, hashing, signing, or validation.

### 1.7 Design Intent and Non-Goals (Informative)

Neural Lock is a risk-reduction mechanism, not a proof system.

Neural Lock:

* does not prove coercion
* does not authenticate identity via biometrics
* does not diagnose medical conditions
* does not replace multisig, timelocks, or operational security
* does not guarantee safety against long-term captivity or determined attackers

Neural Lock provides an operator state predicate intended to reduce immediate coercion success rates and to enable policy-bound constraints and covert compliance mechanisms (decoy wallets, caps, delays) within an existing custody stack.

Neural Lock is opt-in and is expected to be most useful for high-risk users and institutional custody deployments.

---

## 2. Terminology

* Neural Lock: the complete system for human-state-aware policy evidence production
* Neural Attestation (NeuralAttestation): cryptographic statement about current operator state classification with freshness guarantee
* Signal: raw physiological or behavioural measurement (heart rate, skin conductance, etc.)
* NeuralState: coarse classification of operator condition (NORMAL, STRESSED, DURESS, IMPAIRED)
* ClassifierOutcome: classification output that may be AVAILABLE or UNAVAILABLE
* Policy Gate: decision point where NeuralAttestation influences operation authorisation
* Decoy Wallet: alternative wallet presented under duress conditions
* Recovery Capsule: encrypted notification sent when duress is detected
* Break Glass: governed emergency override path requiring guardian quorum and delay
* Training Mode: supervised calibration period for establishing baselines and stress profiles
* Advisory Mode: warning-only mode with no enforcement
* Enforcement Mode: gating mode requiring predicate satisfaction or compensating controls

Key terms from the PQ ecosystem:

* ConsentProof: user authorisation artefact
* Epoch Clock: verifiable time source
* PQHD: Post-Quantum Hierarchical Deterministic Custody
* Multi-Predicate Custody: authorisation requiring multiple conditions
* ZET/ZEB: Zero-Exposure Transaction and Broadcast
---

## 2A. Explicit Dependencies

### Core Dependencies (All Deployments)

| Specification | Minimum Version | Purpose |
|---------------|-----------------|---------|
| PQSEC | ≥ 2.0.3 | Enforcement of operator_state_ok predicate |
| PQSF | ≥ 2.0.3 | Canonical encoding for NeuralAttestation |
| Epoch Clock | ≥ 2.1.0 | Freshness binding for attestations and alignment with PQSEC staleness model |

### Reference Implementation Dependencies (Bitcoin Custody)

| Specification | Minimum Version | Purpose |
|---------------|-----------------|---------|
| PQHD | ≥ 1.2.0 | Bitcoin custody integration and predicate composition |

For non-Bitcoin deployments, PQHD is not required. Consuming systems must provide equivalent predicate composition and enforcement mechanisms appropriate to their domain.

Implementations MAY evaluate using earlier versions, but MUST NOT claim conformance while below the stated minimums.

Neural Lock produces attestation artefacts only. All enforcement is performed by PQSEC.

---

## 3. Architecture

### 3.1 System Overview

Neural Lock produces operator-state evidence consumed by a policy engine and enforced via PQSEC.

```
Bitcoin Wallet
  User UI -> Transaction Builder -> Signing Engine
                    |
                 Policy Engine
                    |
              Neural Lock Layer
        Signal Aggregator -> State Classifier
                    |
             NeuralAttestation (if available)
                    |
           Sensor Interface Layer
     Biometric Sensors + Behavioral Sensors
```

### 3.2 Data Flow

1. Sensors collect signals continuously
2. Signal Aggregator fuses weighted measurements
3. State Classifier determines current outcome
4. NeuralAttestation generated with freshness binding when outcome is AVAILABLE
5. Policy Engine consumes state (not raw signals)
6. Operation authorisation determined by policy and state
7. State transitions logged locally (optional)

### 3.3 Integration Point

Neural Lock sits between intent and signing in the operation flow:

```
User Intent -> Policy Engine -> Neural Lock Gate -> Signing Conditions -> Execution
```

Neural Lock never touches:

* private keys
* seed phrases
* signing operations
* transaction construction

Neural Lock only influences:

* policy decisions
* authorisation thresholds
* capability constraints
* notification triggers

### 3.4 Canonical Encoding Requirements

All Neural Lock artefacts that are hashed, signed, compared, or consumed by PQSEC MUST be encoded using PQSF canonical encoding rules.

Requirements:

1. Canonical encoding MUST use Deterministic CBOR.
2. Floating-point values MUST NOT appear in any canonical artefact.
3. All numeric values MUST be represented as fixed-point integers.
4. Re-encoding a decoded artefact MUST produce byte-identical output.
5. Non-canonical encodings MUST be rejected.

Neural Lock does not define encoding rules independently and fully inherits canonical encoding requirements from PQSF.

Hashing note: All hashes in Neural Lock follow PQSF §9 via the active CryptoSuiteProfile (Tier 2 SHAKE256-256, 32 bytes) unless explicitly stated otherwise.

---

## 4. Signal Sources and State Classification

### 4.1 Signal Categories

#### 4.1.1 Physiological Signals (Primary)

Heart Rate Variability (HRV)

* measurement: inter-beat interval variation
* risk indicator: reduced HRV consistent with elevated stress response
* source: smartwatch, fitness tracker, chest strap
* sampling rate: 1 Hz minimum
* weight (fixed-point): weight_value=350, weight_scale=1000

Skin Conductance (Galvanic Skin Response)

* measurement: electrodermal activity
* risk indicator: elevated conductance consistent with stress response
* source: wearable sensor, phone grip sensor
* sampling rate: 10 Hz
* weight (fixed-point): weight_value=250, weight_scale=1000

Heart Rate Absolute

* measurement: beats per minute
* risk indicator: elevated or suppressed (context-dependent)
* source: any heart rate monitor
* sampling rate: 1 Hz
* weight (fixed-point): weight_value=200, weight_scale=1000

#### 4.1.2 Behavioural Signals (Secondary)

Typing Cadence

* measurement: inter-keystroke timing, error rate
* risk indicator: deviation from baseline pattern
* source: device input subsystem
* weight (fixed-point): weight_value=100, weight_scale=1000

Touch Pressure and Tremor

* measurement: force variance, steadiness
* risk indicator: increased tremor, pressure changes
* source: touchscreen pressure sensors
* weight (fixed-point): weight_value=50, weight_scale=1000

Interaction Timing

* measurement: speed of UI navigation, hesitation
* risk indicator: unusually fast or slow interactions
* source: application event logs
* weight (fixed-point): weight_value=50, weight_scale=1000

### 4.2 Classification Outputs

#### 4.2.1 NeuralState

```rust
pub enum NeuralState {
    NORMAL,
    STRESSED,
    DURESS,
    IMPAIRED,
}
```

IMPAIRED indicates an operator condition in which high-risk decisions are more likely to be unsafe, for example intoxication, acute illness, concussion, severe sleep deprivation, or severe cognitive overload. IMPAIRED is a policy signal, not a medical determination.

#### 4.2.2 ClassifierOutcome

ClassifierOutcome supports unavailable classification without inventing new attestation states.

ClassifierOutcome MUST be one of:

* AVAILABLE: produces a NeuralState and a NeuralAttestation
* UNAVAILABLE: no valid classification for policy use; no NeuralAttestation is produced

When the outcome is UNAVAILABLE, consuming systems MUST apply Section 11.8 (Graceful Degradation).

### 4.3 Robust Deviation Score (MAD-based)

Input: weighted signal vector S with weights W
Baseline: user-specific baseline B

Robust deviation score:

```
D = Σ(w_i * |s_i - median_i| / mad_i)
```

Where:
* `median_i` is the baseline median for signal i (from user-specific baseline B)
* `mad_i` is the baseline MAD (Median Absolute Deviation) for signal i
* `s_i` is the current signal value for signal i
* `w_i` is the fixed-point weight for signal i

MAD clamping requirement:

```
mad_i = max(median(|s_j - median_i|) * MAD_SCALE_FACTOR, 1)
```

MAD_SCALE_FACTOR is deployment-defined but MUST be deterministic and documented for conformance. It MUST be applied consistently within a deployment.

Thresholds:

* thresholds MUST be policy-defined (recommended) or configuration-defined
* thresholds MUST NOT depend on floating point arithmetic

### 4.4 Stress Profile Calibration Requirement

Implementations MUST distinguish:

* normal operating stress (exercise, work stress, daily elevated arousal)
* duress-like stress (coercion-correlated patterns)

This distinction MUST be represented in baseline and threshold calibration inputs (see Section 11.2).

### 4.5 Confidence Representation (Fixed-Point)

Confidence MUST NOT use floating point.

```
confidence = confidence_value / confidence_scale
```

Recommended:

* confidence_scale = 1000
* confidence_value in [1, 1000] for emitted attestations (0 reserved for internal/non-emitted cases)

Confidence semantics:

* confidence represents signal coverage and freshness only
* confidence MUST NOT be interpreted as correctness or likelihood of coercion
* implementations MUST NOT expose "coercion probability" UI

### 4.6 State Persistence and Debounce

* minimum state duration: policy-defined (RECOMMENDED: 5 seconds) (debouncing)
* state transitions require sustained signal change
* DURESS is sticky as defined in Section 7.3

### 4.7 Sensor Failure Handling

Missing signals:

* if primary signals unavailable: reduce confidence, bias toward STRESSED where supported by remaining valid signals
* this fallback behaviour MUST be policy-configurable
* deployments SHOULD require compensating predicates (guardian approval, delays) for meaningful operations when relying on degraded signal sets
* if signals are insufficient to classify: outcome MUST be UNAVAILABLE

Contradictory signals:

* weighted voting among enabled and valid signals
* if confidence below policy minimum due to signal contradiction (not absence): escalate to DURESS only when:
  - sufficient signal coverage exists (primary signals present), AND
  - signals are contradictory or show anomalous patterns, AND
  - spoofing or liveness failure is not detected
* if confidence is low due to signal absence or staleness: outcome MUST be UNAVAILABLE
* low confidence from insufficient evidence MUST NOT map to DURESS

### 4.8 Liveness Detection

Spoofing indicators (non-exhaustive):

* zero variance over a defined window
* abrupt reappearance after prolonged absence
* physiological vs behavioural mismatch

Spoofing classification:

1. classifier outcome MUST be AVAILABLE with NeuralState = DURESS
2. confidence_value MUST be set to the minimum non-zero representable value (policy-default: 1/confidence_scale)
3. a deterministic spoofing_reason code MUST be recorded locally
4. raw signal data MUST NOT be retained

Reason codes are local-only:

* reason codes MUST NOT be included in NeuralAttestation
* reason codes SHOULD be recorded only in encrypted local logs

Sensor liveness proof (optional):
Implementations MAY perform deterministic liveness checks to mitigate replayed or injected sensor streams.

Liveness safety requirements:

1. liveness MUST NOT be required for ordinary spending or routine operations
2. if enabled, liveness MUST be limited to policy-defined high-risk operation classes
3. liveness prompts MUST be consistent with normal-looking wallet UX and MUST NOT explicitly signal "coercion detection"
4. liveness MUST be triggered by operation class policy, not by classification outcome
5. liveness evaluation MUST be deterministic and MUST NOT require network connectivity

To resist spoofing and replayed sensor streams, implementations SHOULD implement multi-signal liveness indicators.

**Recommended indicators (non-exhaustive):**

* **Cross-signal correlation**: verify consistency between independent signal channels (for example physiological vs behavioural timing).
* **Physiological plausibility bounds**: reject values outside deployment-defined bounds.
* **Rate-of-change consistency**: detect unnatural stability (for example zero variance over a defined window) and abrupt discontinuities.

These indicators produce optional evidence only.
PQSEC evaluates all inputs and applies policy.

No liveness check is infallible.
Neural Lock reduces coercion success rates; it does not guarantee detection.

### 4.9 Network Independence (Normative)

Neural Lock classification and NeuralAttestation generation MUST NOT require network connectivity.

Requirements:

1. classification MUST function with no network access
2. attestation generation MUST function with no network access
3. liveness challenges, when implemented, MUST be locally verifiable
4. freshness is determined by the verified Epoch Clock tick input consumed locally, not by external time sources

### 4.10 Power Management (Normative)

When the device enters sleep, suspend, or equivalent power-saving states:

1. active sensor sampling MAY pause
2. upon wake, Neural Lock MUST re-establish fresh signals before producing NeuralAttestations used for operations requiring Neural Lock
3. during sleep, classifier outcome is UNAVAILABLE
4. if an operation requiring Neural Lock is attempted immediately after wake and fresh signals are not yet available, classifier outcome MUST be UNAVAILABLE and Section 11.8 MUST apply

### 4.11 UNAVAILABLE Classification Semantics (Normative)

Neural Lock supports explicit unavailability of operator-state classification.

### 4.11.1 ClassifierOutcome UNAVAILABLE

`UNAVAILABLE` indicates that Neural Lock cannot produce a valid NeuralAttestation due to genuine absence of required signals or operating prerequisites.

Examples (non-exhaustive):

* device sleep or wake transition
* insufficient sensor coverage
* sensor dropout or hardware unavailability
* power state transitions preventing signal stabilization

`UNAVAILABLE` does not assert NORMAL, STRESSED, DURESS, or IMPAIRED. It asserts absence of evaluable evidence only.

### 4.11.2 Attestation Production Rules

1. When ClassifierOutcome is `UNAVAILABLE`, Neural Lock MUST NOT produce a NeuralAttestation.
2. Neural Lock MUST NOT emit a best-effort, stale, or heuristic attestation to avoid UNAVAILABLE.
3. Neural Lock MUST NOT downgrade UNAVAILABLE to any NeuralState.
4. Neural Lock MUST surface UNAVAILABLE explicitly to consuming policy engines.

### 4.11.3 Enforcement Boundary

Neural Lock does not map UNAVAILABLE to permission or refusal.

All enforcement decisions derived from UNAVAILABLE outcomes are performed exclusively by PQSEC according to active policy and operation class.

### 4.11.4 Determinism Requirement

Given identical local signal availability and operating conditions, Neural Lock MUST produce identical ClassifierOutcome results, including identical UNAVAILABLE classifications.

### 4.11.5 Audit and Logging

Local implementations MAY log UNAVAILABLE events for diagnostic or audit purposes.

Such logs:

1. MUST remain local to the device.
2. MUST NOT include raw biometric or behavioural signal data.
3. MUST NOT be exported or treated as enforcement artefacts.
4. MUST NOT include timestamps with sub-tick precision.
5. MUST NOT include signal-specific failure reasons or sensor identifiers.

---

### 4.12 Manual Duress Signal (Normative if Implemented)

### 4.12.1 Purpose

Neural Lock MAY support a holder-issued manual signal to accelerate transition into a coercion posture during time-critical events.

Manual duress signals are evidence only. They grant no authority and cannot enable operations.

### 4.12.2 Signal Types

A conformant implementation MAY support one or more of the following local-only signal types:

* **phrase cue** (speech-to-text local match)
* **gesture cue** (UI sequence, button pattern, touch cadence)
* **hardware cue** (side button pattern, wearable tap pattern)

Implementations MUST NOT require a specific signal type.

### 4.12.3 Default Behaviour

When a valid manual duress signal is detected:

1. The classifier outcome MUST be AVAILABLE.
2. The resulting NeuralState MUST be set to DURESS.
3. `confidence_value` MUST be set to the minimum representable value (e.g. 1/1000) to indicate "manual assertion, not sensor coverage".
4. A NeuralAttestation MUST be generated and bound to the active `session_id` and `intent_hash` when present.

Manual signals MUST be treated as one-way: they can move into DURESS but MUST NOT move out of DURESS.

### 4.12.4 Signal Scope

When a session is active, the manual duress signal MUST be bound to the active `session_id`.

When no session is active, the manual duress signal MUST apply at device scope and MUST govern all subsequent sessions until cleared per §7.3 exit conditions.

### 4.12.5 Duress Exit

Manual duress signals do not define their own exit path. The duress state persists until cleared via the exit conditions defined in §7.3 (manual clearance under sustained NORMAL, guardian-governed clearance, or factory reset).

### 4.12.6 Anti-Coercion Safety Requirements

Because adversaries may observe or compel manual signals:

1. Manual duress signals MUST NOT produce any externally visible acknowledgement distinct from normal behaviour.
2. Manual duress signals MUST NOT display "duress mode enabled" or any equivalent UI indication.
3. Manual signal configuration MUST support deterministic rotation of the phrase/gesture keyed by a local secret and Epoch Clock tick, so observers cannot predict future cues.
4. Implementations MUST NOT store the raw spoken phrase, raw audio, or raw gesture telemetry outside the secure local boundary.

### 4.12.7 Authority Boundary

Manual duress signals:

* MUST NOT permit any operation
* MUST NOT override policy denials
* MUST NOT relax any predicate requirement

They exist solely to increase refusal and enable constrained decoy posture where policy permits.

---

## 5. Policy Integration

### 5.1 NeuralAttestation Schema (Normative)

NeuralAttestation MUST be canonical CBOR.

All fields below are canonical fields.

```rust
pub struct NeuralAttestation {
    pub attestation_version: u16,   // MUST be 1 for this specification
    pub state: NeuralState,

    // Fixed-point confidence: confidence = value / scale
    pub confidence_value: u32,
    pub confidence_scale: u32,

    // Epoch Clock tick value (unit defined by Epoch Clock profile)
    pub timestamp_tick: u64,

    // Freshness window expressed in ticks (same unit as timestamp_tick)
    pub freshness_window_ticks: u64,

    pub signal_sources: Vec<SignalSource>,
    pub device_id: [u8; 32],

    // Binding fields to prevent replay and cross-session reuse
    pub session_id: Option<[u8; 32]>,
    pub intent_hash: Option<[u8; 32]>,

    // Evidence producer integrity binding (see §5.9)
    pub classifier_build_hash: [u8; 32],

    // Profile indirection (no hard-coded algorithms)
    pub suite_profile: String,
    pub signature: Vec<u8>,
}
```

### 5.2 Intent and Session Binding (Strongly Recommended)

To prevent attestation replay and cross-session reuse, implementations SHOULD include binding fields:

**session_id:**

* Optional [u8; 32] binding to current authentication/application session
* SHOULD be included for all session-scoped operations
* MUST be derived deterministically from session context
* MUST change on session boundaries (re-authentication, app restart, time expiry)

**intent_hash:**

* Optional [u8; 32] binding to specific operation intent
* SHOULD be included for Authoritative operations (custody, signing, high-value)
* MAY be omitted for continuous monitoring scenarios
* When present, MUST be the canonical hash of the operation being authorized

**PQSEC Integration Requirement:**

When session_id or intent_hash is present in NeuralAttestation:

* PQSEC MUST verify binding matches current operation context
* Binding mismatch MUST result in operator_state_ok = FALSE (per PQSEC §8A.1 ternary predicate model). A binding mismatch is definitive evidence of invalid state, not absence of evidence, and therefore maps to FALSE rather than UNAVAILABLE.
* PQSEC MUST NOT accept NeuralAttestation across session boundaries without explicit policy

When binding fields are absent:

* PQSEC MUST treat NeuralAttestation as session-scoped by default
* PQSEC MUST implement replay protection via freshness + decision_id tracking
* Deployments accepting unbound attestations MUST document the security trade-off

### 5.2A ClassifierOutcome to PredicateResult Mapping (Normative)

The following table defines the normative mapping from Neural Lock ClassifierOutcome values to PQSEC predicate results for `operator_state_ok`:

| ClassifierOutcome | operator_state_ok PredicateResult | Notes |
|-------------------|-----------------------------------|-------|
| NORMAL            | TRUE                              | Operator state verified nominal |
| STRESSED          | TRUE (with constraints per policy) | Policy may restrict operation classes |
| DURESS            | FALSE                             | Definitive evidence of compromised state |
| IMPAIRED          | FALSE                             | Definitive evidence of compromised state |
| UNAVAILABLE       | UNAVAILABLE                       | No evidence available; fail-closed per PQSEC §8A |

STRESSED mapping to TRUE with constraints means: the predicate evaluates TRUE, but active policy MAY restrict which operation classes proceed when STRESSED is the underlying classification. The constraint mechanism is policy-defined. PQSEC does not grant authority based on Neural Lock output; the predicate result informs the enforcement decision.

### 5.3 Tick Units Requirement

freshness_window_ticks is expressed in the same units as the Epoch Clock tick. Implementations MUST NOT treat tick values as seconds unless the Epoch Clock profile defines that mapping.

### 5.4 suite_profile Registry Requirements (Minimal)

suite_profile MUST be a stable identifier for the cryptographic suite used to sign NeuralAttestation.

Requirements:

1. suite_profile MUST be stable across implementations within a deployment
2. suite_profile identifiers MUST be versioned
3. suite_profile MUST determine at minimum: signature algorithm family and parameters, public key format and verification rules, and domain separation rules if any
4. unknown suite_profile values MUST cause rejection

### 5.5 device_id Lifecycle and Privacy Requirements

device_id identifies the attestation-producing device within a deployment.

Requirements:

1. device_id MUST be stable for the duration of an enrollment
2. device_id SHOULD be derived to avoid global cross-deployment correlation
3. implementations SHOULD support device_id rotation by re-enrollment
4. device_id MUST NOT embed plaintext hardware serial numbers or globally unique identifiers

### 5.6 SignalSource Schema and Ordering (Normative)

SignalSource identifies which signal channels were used, without revealing raw data.

```rust
pub struct SignalSource {
    pub source_id: String,     // deterministic identifier
    pub source_type: String,   // e.g. "heart_rate", "hrv", "gsr", "typing"
    pub device_ref: [u8; 32],  // sensor or subsystem identifier (pseudonymous)
    pub last_update_tick: u64, // Epoch Clock tick of latest included sample
}
```

Ordering requirements:

1. signal_sources MUST be sorted deterministically by source_id using lexicographic byte order over UTF-8
2. re-encoding after decode MUST preserve identical ordering

### 5.7 Signature Input

The signature MUST be computed over the canonical CBOR encoding of the NeuralAttestation with signature omitted.

attestation_version MUST be included in the signed payload.

### 5.8 Policy Evaluation Order

Policy evaluation order for operations that require Neural Lock:

1. kill switch
2. classifier outcome evaluation
3. consent existence (ConsentProof)
4. policy caps and constraints
5. custody quorum and role constraints
6. signature validation and execution

---

### 5.9 Evidence Producer Integrity Binding (Normative)

#### 5.9.1 Purpose

This section binds Neural Lock attestations to a specific, auditable classifier build to prevent compromised or substituted evidence producers from emitting structurally valid but semantically malicious attestations.

#### 5.9.2 Classifier Build Hash

Each NeuralAttestation MUST include a `classifier_build_hash` field.

```
classifier_build_hash: bstr(32)
```

Definition:

```
classifier_build_hash = HASH( canonical_bytes_of_classifier_binary )
```

Where `HASH` is the active hash function defined by the CryptoSuiteProfile (see PQSF §9).

Rules:

1. The input MUST be the exact executable or WASM binary used for classification.
2. Dynamic libraries, model weights, and configuration files that influence classification MUST be included in the hashed material or referenced by an included manifest hash.
3. The hash MUST be stable across identical builds (reproducible-build compatible).

#### 5.9.3 Manifest Option (Optional)

Implementations MAY use a manifest for multi-component classifiers:

```
classifier_manifest = {
  binary_hash: bstr,
  model_weights_hash: bstr,
  config_hash: bstr
}
classifier_build_hash = HASH( canonical(classifier_manifest) )
```

`canonical(classifier_manifest)` MUST mean PQSF deterministic CBOR encoding.

#### 5.9.4 Authority Boundary

`classifier_build_hash` is evidence only. It MUST NOT grant authority or override enforcement. It exists solely for verification and pinning by consuming specifications (see PQSEC §22A).

---

### 5.10 Emission Discipline (Normative)

#### 5.10.1 Scope

This section constrains when Neural Lock may emit NeuralAttestations and at what frequency to prevent behavioural fingerprinting from state telemetry.

#### 5.10.2 Operation-Scoped Emission Only

Neural Lock MUST emit NeuralAttestations only in the context of an operation attempt that requires operator-state evidence.

1. Continuous background emission is prohibited.
2. Periodic telemetry emission is prohibited.
3. NeuralAttestation production MUST be triggered only by an operation-class gate.

#### 5.10.3 UNAVAILABLE Non-Distinguishability at External Boundary

UNAVAILABLE behaviour MUST NOT create externally observable fingerprints.

1. If ClassifierOutcome is UNAVAILABLE, Neural Lock MUST produce no attestation.
2. Consuming components MUST map UNAVAILABLE to the same external refusal surface as any other refusal cause (see PQSEC §28A).
3. Neural Lock MUST NOT export or log distinct UNAVAILABLE reason codes outside the local device.

#### 5.10.4 Rate Limits

Neural Lock MUST apply a strict emission rate limit:

* maximum one attestation per `intent_hash` per session
* maximum one attestation per `session_id` per operation attempt

Attempts to request multiple attestations for the same intent MUST return the prior attestation if still fresh, or return UNAVAILABLE if freshness cannot be established without re-sampling.

If an implementation caches and replays a prior attestation for the same (session_id, intent_hash) within freshness, the cache MUST be scoped to the local signing domain. Cache persistence across restarts is RECOMMENDED. If cache integrity cannot be guaranteed, the implementation MUST fail closed by returning UNAVAILABLE rather than emitting a second attestation.

#### 5.10.5 Device Identifier Privacy

`device_id` MUST be derived to prevent cross-deployment correlation.

1. `device_id` MUST be scoped to a single deployment or wallet domain.
2. `device_id` rotation SHOULD be supported by re-enrolment.
3. `device_id` MUST NOT include hardware serial numbers or platform identifiers.

Note: These rules extend §5.5 (device_id Lifecycle and Privacy Requirements) with explicit correlation-resistance requirements.

#### 5.10.6 Export Controls

Any export of Neural Lock artefacts MUST be governed by a ReceiptExportPolicy (PQSF §17A). Default behaviour MUST be local-only storage.

---

## 6. Bitcoin Integration Patterns

### 6.1 Integration with PSBT Workflow

1. user initiates spend
2. wallet constructs PSBT
3. policy engine requests classifier outcome and (if available) NeuralAttestation
4. classifier returns:

   * AVAILABLE with a NeuralState and NeuralAttestation, or
   * UNAVAILABLE with no NeuralAttestation
5. policy engine evaluates constraints:

   * AVAILABLE and state NORMAL: proceed
   * AVAILABLE and state STRESSED: apply constraints
   * AVAILABLE and state DURESS: route to decoy constraints
   * AVAILABLE and state IMPAIRED: deny or constrain per policy
   * UNAVAILABLE: follow Section 11.8
6. if authorised, PSBT signed
7. broadcast or hold per policy

### 6.2 Decoy Wallet Architecture

Primary wallet:

* majority of funds
* requires NORMAL (or stricter) for high-risk operations

Decoy wallet:

* plausible low-value funds
* separate derivation path
* constraints: caps, delays, and broadcast discipline

### 6.3 Decoy Derivation Path and Marker

This specification defines a recommended hardened marker index 99' for the decoy branch.

To reduce social predictability, deployments MAY select a different hardened marker index, provided:

1. the marker is policy-defined and deterministic within the deployment
2. primary and decoy paths remain strictly separated
3. the chosen marker is documented for conformance within the deployment

### 6.4 Bitcoin Integration Context (Informative)

Neural Lock integrates with Bitcoin custody systems without constraining output types or script construction. Typical deployment contexts include:

* **Timelocks:** CLTV and CSV for delay enforcement
* **Output types:** P2WPKH, P2WSH, P2TR (no restriction)
* **Key derivation:** BIP32 for wallet structure  
* **Transaction format:** PSBT v2 for policy composition

Neural Lock does not define spend paths, script templates, or output constraints. Policy engines determine these based on operator_state_ok evaluation.

### 6.5 Execution Revelation Gating (ZET)

When used with ZET execution patterns:

1. Neural Lock evaluation MUST complete before execution revelation
2. Execution MUST NOT proceed unless operator_state_ok is OK
3. if state enters DURESS or IMPAIRED after intent formation, execution MUST abort
4. UNAVAILABLE outcome MUST follow Section 11.8

---

## 6A. Beyond Bitcoin: General Application Domains (Informative)

While this specification focuses on Bitcoin custody as the primary application, Neural Lock's architecture is domain-agnostic. The operator_state_ok predicate may be applied to any authorization context where coercion resistance is valuable.

### 6A.1 Applicable Domains

**Multi-Chain Cryptocurrency Custody**

* Ethereum, Solana, and other blockchain ecosystems
* Cross-chain bridge operations
* DEX trading approvals
* NFT transfers and marketplace operations
* Staking/unstaking operations

**Traditional Finance**

* Wire transfer authorization
* Large withdrawal approvals
* Account closure or beneficiary changes
* Trading limit modifications
* Vault access in institutional custody

**Enterprise Systems**

* Root or admin access to production systems
* Data deletion operations (GDPR, retention policies)
* Cryptographic key generation and distribution
* Certificate issuance and revocation
* Backup restoration authorization

**Healthcare**

* Controlled substance prescription authorization
* Life support system modifications
* Access to sensitive patient records
* Organ donation consent verification
* Medical device parameter changes

**Industrial Control Systems**

* Safety system override authorization
* Emergency shutdown procedures
* Critical process parameter changes
* Hazardous material handling approvals

**Legal and Notary Services**

* Document signing under potential duress
* Power of attorney execution
* Will modifications
* Property transfer authorizations

### 6A.2 Integration Pattern for Non-Bitcoin Domains

When applying Neural Lock outside Bitcoin custody:

**1. Define Operation Classification**

Map your operations to Neural Lock operation classes:
* **Authoritative:** Operations with irreversible consequences (transfers, deletions, system changes)
* **Non-Authoritative:** Operations that can be reversed or are low-risk

**2. Map States to Domain Policies**

Define domain-appropriate responses for each NeuralState:

* **NORMAL:** Standard authorization flow
* **STRESSED:** Apply constraints (approval requirements, delays, reduced limits, enhanced logging)
* **DURESS:** Route to safe alternative (analogous to decoy wallet) or require additional verification
* **IMPAIRED:** Deny high-risk operations or require guardian/supervisor approval

**3. Implement Intent Binding**

For Authoritative operations:
* Compute intent_hash from operation parameters
* Include in NeuralAttestation for binding verification
* Prevents attestation replay across different operations

**4. Define Session Boundaries**

Establish clear session lifecycle:
* Authentication events that start sessions
* Events that terminate sessions (logout, timeout, re-authentication required)
* Generate session_id at session start
* Include in NeuralAttestation for session binding

**5. Use operator_state_ok as Policy Predicate**

Integrate Neural Lock evaluation into authorization logic:
```
if operation.is_high_risk():
    attestation = neural_lock.get_current_attestation()
    
    if attestation is None:
        return handle_unavailable(operation)
    
    result = policy_engine.evaluate(
        operation=operation,
        operator_state_ok=attestation.validate(),
        other_predicates=...
    )
    
    return enforcement_engine.enforce(result)
```

**6. Defer Enforcement to Domain Policy Engine**

Neural Lock produces evidence only. The consuming system:
* Defines what each state means for each operation type
* Implements enforcement (approval, delay, refusal, escalation)
* Logs decisions and state transitions
* Handles graceful degradation when attestations unavailable

### 6A.3 Domain-Specific Considerations

**DURESS Handling**

Bitcoin uses decoy wallets. Other domains require different safe actions:
* Banking: Route to fraud prevention team
* Healthcare: Require in-person verification
* Enterprise: Trigger silent alarm, require second approver
* Industrial: Initiate supervised mode, notify safety officer

**STRESSED Handling**

Adjust constraints based on domain risk:
* Financial: Reduce transaction limits, require delay
* Medical: Require peer review for critical decisions
* Industrial: Enable additional safety interlocks
* Enterprise: Require break-glass approval for destructive operations

**IMPAIRED Handling**

May require stronger constraints than financial systems:
* Medical: Absolute denial for critical procedures
* Industrial: Lock out safety-critical controls
* Legal: Postpone irrevocable signing
* Enterprise: Suspend privileged access pending review

**Training Baselines**

Calibration periods may vary by domain:
* Financial: 7-14 days (standard)
* Healthcare: 30 days (higher reliability required)
* Industrial: Shift-specific baselines (different operational contexts)
* Enterprise: Role-specific baselines (admin vs developer)

**Freshness Windows**

Adjust based on operation latency requirements:
* Banking: 60-300 seconds (human-paced)
* Healthcare: 300-600 seconds (longer deliberation)
* Industrial: 10-30 seconds (rapid response)
* Enterprise: 120-600 seconds (varies by operation)

### 6A.4 Reference Integration Points

**Section 10.1 (Generalised Security Gating)** provides normative requirements for non-Bitcoin implementations.

**Annex I (Generic Operator State Gating Interface)** provides reference pseudocode applicable to any domain.

**Annex J (Quick Start Guide)** applies to all domains, not just Bitcoin.

### 6A.5 Important Boundaries

Neural Lock does NOT define:
* Domain-specific authorization logic
* What operations are "high-risk" in your domain
* How to respond to DURESS in your context
* Enforcement mechanisms or refusal semantics
* Logging or audit requirements

Neural Lock provides operator state evidence. Your policy engine defines what to do with it.

### 6A.6 Implementation Checklist for Non-Bitcoin Domains

- [ ] Classify operations as Authoritative vs Non-Authoritative
- [ ] Define NeuralState → Policy mappings for each operation class
- [ ] Implement intent_hash computation for Authoritative operations
- [ ] Define session boundaries and session_id generation
- [ ] Integrate operator_state_ok into authorization logic
- [ ] Define graceful degradation behavior for UNAVAILABLE
- [ ] Implement domain-appropriate "safe actions" for DURESS
- [ ] Establish training baseline requirements
- [ ] Set freshness windows appropriate to operation latency
- [ ] Configure confidence thresholds per operation class
- [ ] Test with simulated stress and sensor failure scenarios
- [ ] Document security trade-offs and policy decisions

---

## 7. Security Properties

### 7.1 Core Security Guarantees

* non-bypassability
* fail-safe default
* determinism
* revocation discipline
* auditability

### 7.2 Lockout Interaction

Neural Lock MUST NOT implement lockout logic. PQSEC defines lockout semantics.

### 7.3 Sticky DURESS and Exit Conditions (Normative)

DURESS MUST NOT clear automatically.

Exit from DURESS MUST require one of:

1. manual clearance under sustained NORMAL window, requiring authentication and audit logging
2. PQSEC-governed clearance requiring guardian participation
3. factory reset or re-enrolment requiring explicit physical or governed authorisation

For exit condition (1), sustained NORMAL window duration MUST be policy-defined and SHOULD be at least 5 minutes continuous NORMAL.

Time-based decay MAY be used only if explicitly permitted by policy and MUST be deterministic and audit-logged.

---

## 8. Privacy Protections

### 8.1 Data Minimisation

Raw signal data MUST NOT be stored, transmitted, or logged.

Only coarse state and fixed-point confidence are exported.

### 8.2 Local Processing

Classification and attestation generation occur locally.

### 8.3 Encrypted Logs (Optional)

If logs are retained:

* they MUST be encrypted
* they MUST NOT include raw biometrics
* they SHOULD support export and deletion controls

### 8.4 Legal Neutrality and No Medical Claims

Neural Lock makes no claims regarding legal capacity, consent validity, or medical status. It produces security evidence only.

Neural Lock does not diagnose medical conditions. The IMPAIRED state indicates only that physiological and behavioural patterns resemble those correlated with impaired decision-making in security contexts.

---

## 9. Threat Model

### 9.1 Threats Mitigated

| Threat                    | Mitigation                      | Effectiveness | Detection Time     |
| ------------------------- | ------------------------------- | ------------- | ------------------ |
| Physical coercion         | decoy wallet plus state gating  | High          | Seconds            |
| Social engineering stress | cooling-off constraints         | Medium-High   | Seconds to minutes |
| Impairment                | deny or constrain when IMPAIRED | High          | Seconds to minutes |
| Delegation abuse          | revoke or suspend delegations   | High          | Seconds            |
| Panic transactions        | delays when STRESSED            | Medium        | Seconds            |

Detection time is deployment-dependent and MUST be documented.

### 9.2 Threats Not Mitigated

| Threat                        | Why Not Mitigated           | Recommended Defence                             |
| ----------------------------- | --------------------------- | ----------------------------------------------- |
| Long-term coercion            | patience and captivity      | supervised mode, guardian monitoring            |
| Full device compromise        | classifier runs on device   | PQSEC runtime attestation plus secure boot plus isolation |
| Correlated guardian coercion  | guardians coerced together  | quorum diversity, independent guardians, delays |
| Key extraction                | not a key-protection system | hardware isolation plus PQHD quorum             |
| Post-broadcast quantum attack | Bitcoin limitation          | SEAL plus ZET plus ZEB discipline              |

### 9.3 Honest Limitations

Neural Lock does not prove coercion. It reduces risk by detecting anomalous operator states correlated with elevated attack likelihood and applying fail-safe policy constraints.

### 9.4 Guardian Selection Guidance (Informative)

To reduce risk of correlated compromise, guardians SHOULD be selected with diversity in mind:

- **Geographic distribution**: guardians in separate physical regions
- **Jurisdictional separation**: guardians under different legal regimes
- **Technical heterogeneity**: diverse hardware, OS, and network providers

This section is guidance only and does not modify quorum enforcement semantics, which are defined and enforced by PQSEC.

---

## 10. Integration with PQ Ecosystem

Neural Lock contributes exactly one predicate:

```
operator_state_ok
```

operator_state_ok evaluation:

* OK: NeuralAttestation is present and all required validation steps succeed for the operation class policy
* NOT_OK: NeuralAttestation is present but fails one or more required validation steps
* UNAVAILABLE: no NeuralAttestation is present because classifier outcome is UNAVAILABLE

**PQSEC predicate mapping:** When consumed by PQSEC's ternary predicate model (§8A.1), Neural Lock results map as follows: OK → TRUE, NOT_OK → FALSE, UNAVAILABLE → UNAVAILABLE. This mapping is normative. Consuming enforcement systems MUST treat NOT_OK (FALSE) as predicate failure. UNAVAILABLE MUST be handled via Section 11.8 (Graceful Degradation) and PQSEC §8A tolerance rules.

Validation requirements for OK:

1. NeuralAttestation is present
2. canonical encoding validation succeeds
3. signature verification succeeds under suite_profile
4. attestation_version is supported
5. freshness is within freshness_window_ticks
6. state satisfies policy-required state
7. confidence meets policy minimum

### 10.1 Generalised Security Gating

Neural Lock may be used as an operator state predicate for any operation that requires coercion-resilient risk reduction.

Neural Lock remains non-authoritative in all domains. It produces NeuralAttestation evidence only. All enforcement and refusal semantics MUST remain in the consuming enforcement system.

When used outside Bitcoin custody, consuming systems MUST:

1. define the operation class (Authoritative or Non-Authoritative)
2. define operator state requirements (required_state, minimum confidence)
3. verify session_id and intent_hash bindings when present in NeuralAttestation; consuming systems MUST reject attestations with binding mismatches
4. treat missing, stale, invalid, or unsupported NeuralAttestation as NOT_OK or UNAVAILABLE, and handle UNAVAILABLE via compensating controls or refusal

---

## 11. Implementation Requirements and Guidance

### 11.1 Hardware Requirements (Guidance)

Minimum viable:

* single heart rate sensor, 1 Hz
* deterministic staleness handling
* fail-safe behaviour on sensor loss

Recommended:

* multi-sensor fusion
* hardware-backed device identity
* isolated classifier execution where available

### 11.2 Training and Adaptation

Training Mode MUST include:

1. minimum 7-day baseline
2. user-marked normal stress calibration points
3. optional supervised stress-profile calibration
4. slow-moving adaptation if enabled
5. no adaptation during any DURESS history window
6. Deployments MUST document their baseline calibration methodology and threshold selection rationale as part of their conformance statement.

### 11.3 Accessibility and Physiological Diversity

Implementations MUST support:

* user-specific baselines
* signal exclusion
* conservative weighting
* compensating predicates instead of reduced security

Behavioural-only mode:

* SHOULD be disabled by default
* MUST require explicit acknowledgement of reduced security guarantees
* MUST require compensating predicates under enforcement mode

### 11.4 Baseline Reset Governance (Required)

Baseline reset MUST require guardian quorum and/or enforced delay, and MUST be audit-logged.

### 11.5 Conformance Testing (Required)

Implementations claiming conformance MUST pass Annex E test scenarios.

### 11.6 Break Glass Emergency Override (Optional)

Break Glass MUST be governed by policy, require guardian quorum and delay, and MUST be rate-limited and audit-logged.

### 11.7 Deployment Phases (Recommended)

* Phase 1: advisory only
* Phase 2: low-value enforcement
* Phase 3: full enforcement

### 11.8 Graceful Degradation (Normative)

When Neural Lock cannot produce a valid NeuralAttestation due to sensor failure, calibration expiry, or power management transitions:

1. Advisory mode MAY continue with explicit user-visible warnings and MUST NOT claim protection.
2. Enforcement mode MUST require compensating predicates (for example guardian approval, longer delays, reduced limits) or MUST refuse.
3. Implementations MUST NOT silently downgrade security.
4. Any degradation state MUST be recorded as an audit event.

**PQSEC Integration:** When NeuralAttestation cannot be produced, `operator_state_ok` evaluates to UNAVAILABLE per PQSEC §8A ternary predicate model. PQSEC applies fail-closed mapping: Authoritative operations MUST deny; NonAuthoritative operations MUST deny unless explicit policy permits continuation with compensating controls per PQSEC §8A.5.

Recommended baseline survivability policy (informative):

* under UNAVAILABLE, allow only constrained, non-catastrophic operations (for example decoy access, strict caps, enforced delays, and guardian notification), and require guardians for high-value actions.

### 11.9 ReceiptEnvelope Alignment (Normative)

NeuralAttestation MAY be transported as ReceiptEnvelope type `neural_lock.operator_attestation` using the `neural_lock.*` namespace registered in PQSF Annex W §W.6.

Implementations MUST accept NeuralAttestation in ReceiptEnvelope format when present. Implementations MAY accept raw canonical CBOR NeuralAttestation when policy permits, to support minimal deployments without full ReceiptEnvelope infrastructure.

When transported as ReceiptEnvelope, the NeuralAttestation MUST be the `body` field. All ReceiptEnvelope rules (canonical encoding, signature, authority boundary) apply per PQSF Annex W.

ReceiptEnvelope transport does not alter Neural Lock's authority boundary: `neural_lock.operator_attestation` is evidence only and MUST NOT grant authority.

### 11.10 Training Mode Security Boundary (Normative)

Training mode defines the baseline used for operator state classification and is security-critical.

**Rules:**

1. Training mode MUST be explicitly activated and MUST require the same authorisation level as a policy change.
2. Training mode MUST NOT be entered during, or immediately following, an Authoritative operation failure, lockout, or duress classification.
3. Baseline establishment SHOULD span multiple sessions across distinct time periods (policy-default: ≥ 3 sessions over ≥ 72 hours).
4. Transition from training mode to production mode MUST be treated as an Authoritative operation by the consuming enforcement system and MUST be recorded as a custody-relevant event.
5. Baseline reset or retraining MUST follow the same controls as initial training.

**Retraining frequency control:**

6. Baseline reset and retraining frequency MUST be policy-governed.
7. Deployments SHOULD enforce a minimum interval between retraining events (policy-default: ≥ 30 days).
8. Retraining attempts within the minimum interval MUST be refused unless an explicit policy exception is satisfied (for example, device replacement, verified baseline corruption, or governance-approved reset).
9. Retraining events MUST be recorded as custody-relevant events.

Training artefacts grant no authority and MUST NOT relax enforcement semantics.

### 11.11 Operator Variability and Accommodation (Normative)

Neural Lock baseline training is designed to accommodate operator-specific physiological and behavioural variation.

Implementations MUST account for baseline variability arising from disability, neurodivergence, chronic health conditions, medication, or known high-variance states.

**Rules:**

1. Training mode MUST capture the operator's typical range of states, including known high-variation periods.
2. Implementations SHOULD support configurable sensitivity thresholds per state transition.
3. Deployments SHOULD NOT enable Neural Lock enforcement without a completed baseline training period representative of the operator's normal conditions.
4. Operators MUST be informed of Neural Lock's role and MUST retain the ability to adjust or disable it via policy.

This is a safety and availability requirement, not a relaxation of enforcement.

---

## 12. Comparison to Existing Solutions (Informative)

Duress PINs, timelocks, and multisig each fail under common coercion patterns or operational realities. Neural Lock supplies a covert operator-state predicate and keeps enforcement external, preserving Bitcoin’s trust model.

---

## 13. Future Work

13.1 Multi-Device Synchronisation (Optional)

* primary and secondary devices
* disagreement handling
* evidence aggregation

Other future work:

* formal state-machine proofs
* deterministic ML classifiers
* cognitive challenge extensions
* dedicated wearable hardware

---

## 14. References

* PQSF
* PQSEC
* PQHD
* Epoch Clock
* ZET
* ZEB
* PQAI
* Bitcoin BIPs: 32, 174, 341, 370

---

## 15. Annexes

## Annex A State Classification Pseudocode (Normative)

Definitions:

* SCALE = 1000
* WEIGHT_SCALE = 1000
* CONF_SCALE = 1000
* MAD_SCALE_FACTOR is deployment-defined, deterministic, and documented

```pseudocode
const SCALE = 1000
const WEIGHT_SCALE = 1000
const CONF_SCALE = 1000

function classify(signals, baseline, policy, current_tick):
    fresh = filter_fresh(signals, policy.max_age_ticks, current_tick)

    if count(fresh) < policy.min_signals:
        return { outcome: "UNAVAILABLE", reason: "insufficient_signals" }

    if detect_spoofing(fresh, policy, current_tick):
        return { outcome: "AVAILABLE", state: DURESS, conf_value: 1, conf_scale: CONF_SCALE, reason: "spoofing_detected" }

    deviation = 0

    for s in fresh:
        b = baseline.lookup(s.type)
        if b is null:
            continue

        mad = max(b.mad_raw * max(policy.mad_scale_factor, 1), 1)

        diff = abs(s.value - b.median)
        z_scaled = (diff * SCALE) / mad
        deviation += (z_scaled * b.weight_value) / max(b.weight_scale, 1)

    (conf_value, conf_scale) = compute_confidence(fresh, baseline, policy, current_tick)

    if detect_impairment(fresh, baseline, policy):
        return { outcome: "AVAILABLE", state: IMPAIRED, conf_value: conf_value, conf_scale: conf_scale, reason: "impairment_detected" }

    if deviation < policy.normal_threshold:
        return { outcome: "AVAILABLE", state: NORMAL, conf_value: conf_value, conf_scale: conf_scale, reason: "within_normal_threshold" }

    if deviation < policy.stress_threshold:
        return { outcome: "AVAILABLE", state: STRESSED, conf_value: conf_value, conf_scale: conf_scale, reason: "within_stress_threshold" }

    return { outcome: "AVAILABLE", state: DURESS, conf_value: conf_value, conf_scale: conf_scale, reason: "above_duress_threshold" }
```

Reason codes are local-only and MUST NOT be included in NeuralAttestation.

## Annex B Spoofing and Liveness Detection Pseudocode (Reference)

```pseudocode
function detect_spoofing(signals, policy, current_tick):
    if detect_zero_variance(signals, policy):
        return true

    if detect_phys_behavior_mismatch(signals, policy):
        return true

    if policy.liveness_required_for_operation:
        if not pass_liveness(policy, current_tick):
            return true

    return false
```

## Annex C NeuralAttestation Construction and Signing (Reference)

```pseudocode
function build_attestation(classifier_result, policy, device, current_tick, session_id_opt, intent_hash_opt, classifier_build_hash):
    if classifier_result.outcome != "AVAILABLE":
        return null

    att = {
        attestation_version: 1,
        state: classifier_result.state,
        confidence_value: classifier_result.conf_value,
        confidence_scale: classifier_result.conf_scale,
        timestamp_tick: current_tick,
        freshness_window_ticks: policy.freshness_window_ticks,
        device_id: device.device_id,
        signal_sources: classifier_result.signal_sources,

        # Binding fields (optional but normative when used)
        session_id: session_id_opt,
        intent_hash: intent_hash_opt,

        # Evidence producer integrity binding (required)
        classifier_build_hash: classifier_build_hash,

        suite_profile: policy.suite_profile
    }

    bytes = canonical_cbor_encode(att)
    sig = sign_with_profile(policy.suite_profile, device.signing_key, bytes)
    att.signature = sig
    return att
```

## Annex D PSBT Integration and Proprietary Field Versioning (Reference)

```pseudocode
// Key: 0xFC | "neural_lock" | 0x01
function attach_neural_lock_to_psbt(psbt, attestation):
    key = proprietary_key(prefix=0xFC, ns="neural_lock", ver=0x01)
    psbt.proprietary[key] = canonical_cbor_encode(attestation)
    return psbt
```

If the attestation is wrapped in ReceiptEnvelope before embedding, the ReceiptEnvelope.type MUST be `neural_lock.operator_attestation` and the ReceiptEnvelope.body MUST be the NeuralAttestation object.

**Field Content Requirement:**

PSBT proprietary field value MUST be the canonical CBOR bytes of NeuralAttestation as defined in Section 5.1, encoded according to PQSF canonical encoding rules (Section 3.4).

When consuming NeuralAttestation from PSBT:
1. Extract proprietary field value
2. Decode as canonical CBOR
3. Verify canonical encoding (re-encoding produces byte-identical output)
4. Validate NeuralAttestation per Section 10 requirements

## Annex E Conformance Test Scenarios (Normative)

This annex defines required test scenarios. Implementations claiming conformance MUST handle these cases correctly and MUST produce the specified outcomes. Future versions of this specification MAY provide concrete test vectors with canonical CBOR bytes.

E.1 Valid NORMAL Attestation

* Expected: operator_state_ok = OK if policy requires NORMAL and confidence meets minimum

E.2 Valid DURESS Attestation

* Expected: operator_state_ok = NOT_OK unless policy explicitly allows DURESS for this operation class (for example decoy access)

E.3 Invalid Unsupported Version

* Expected: operator_state_ok = NOT_OK

E.4 Invalid Stale Attestation

* Expected: operator_state_ok = NOT_OK

E.5 Invalid Non-Canonical Encoding

* Expected: operator_state_ok = NOT_OK

E.6 Invalid Low Confidence

* Expected: operator_state_ok = NOT_OK

E.7 Spoofing Zero Variance

* Expected: classifier outcome AVAILABLE with state DURESS and conf_value = 1

E.8 Sensor Loss Insufficient Signals

* Expected: classifier outcome UNAVAILABLE; no NeuralAttestation produced; operation handled via Section 11.8

E.9 Privacy Test No Raw Signal Leakage

* Expected: no raw signals stored, transmitted, or logged

## Annex F Decoy Wallet Derivation and Constraints (Reference)

```pseudocode
primary_path = m / purpose' / coin' / account' / 0' / index
decoy_path   = m / purpose' / coin' / account' / decoy_marker' / index
```

decoy_marker is policy-defined per deployment (recommended 99).

## Annex G Training Mode and Baseline Computation (Informative)

Baseline computation uses deterministic median and deterministic MAD with scaling and clamping.

## Annex H Break Glass Flow (Informative)

Break Glass is a policy-governed override requiring guardian quorum and enforced delay, and must be rate-limited and audit-logged.

## Annex I Generic Operator State Gating Interface (Informative)

### I.1 Core Interface

```pseudocode
OperationIntent = {
  operation_type: string,
  operation_class: "Authoritative" | "NonAuthoritative",
  intent_hash: bytes32,
  session_id: bytes32 | null,
  exporter_hash: bytes | null,
  issued_tick: uint64,
  expiry_tick: uint64
}
```

`bytes32` denotes a fixed 32-byte value. When present, session_id and intent_hash MUST match the 32-byte fields in NeuralAttestation (§5.1).

OperatorStateRequirement = {
  required_state: "NORMAL" | "STRESSED" | "DURESS" | "IMPAIRED",
  min_confidence_value: uint32,
  min_confidence_scale: uint32
}

function gate_operation(intent, requirement, attestation, current_tick):
    if attestation is null:
        return "UNAVAILABLE"

    if attestation.attestation_version != 1:
        return "NOT_OK"

    if not verify_attestation_signature(attestation):
        return "NOT_OK"

    if current_tick - attestation.timestamp_tick > attestation.freshness_window_ticks:
        return "NOT_OK"

    if not state_satisfies(attestation.state, requirement.required_state):
        return "NOT_OK"

    if not confidence_satisfies(attestation, requirement):
        return "NOT_OK"
    
    # Verify binding fields if present
    if attestation.session_id is not null:
        if intent.session_id != attestation.session_id:
            return "NOT_OK"
    
    if attestation.intent_hash is not null:
        if intent.intent_hash != attestation.intent_hash:
            return "NOT_OK"

    return "OK"
```

### I.2 Domain-Specific Integration Examples

#### I.2.1 Bitcoin Custody (Primary Use Case)

```pseudocode
operation_types = {
  "btc_spend": "Authoritative",
  "btc_sign": "Authoritative", 
  "address_generation": "NonAuthoritative"
}

policy_rules = {
  "btc_spend": {
    NORMAL: { action: "allow", constraints: none },
    STRESSED: { action: "allow", constraints: [delay_30s, cap_0.1_btc] },
    DURESS: { action: "route_to_decoy", constraints: [cap_0.01_btc] },
    IMPAIRED: { action: "deny", reason: "cognitive_impairment" },
    UNAVAILABLE: { action: "require", compensating: [guardian_approval] }
  }
}
```

#### I.2.2 Banking Wire Transfer

```pseudocode
operation_types = {
  "wire_transfer": "Authoritative",
  "account_inquiry": "NonAuthoritative"
}

policy_rules = {
  "wire_transfer": {
    NORMAL: { action: "allow", constraints: none },
    STRESSED: { action: "allow", constraints: [delay_120s, notify_fraud_team] },
    DURESS: { action: "route_to_fraud_prevention", silent_alert: true },
    IMPAIRED: { action: "deny", reason: "operator_state" },
    UNAVAILABLE: { action: "require", compensating: [in_person_verification] }
  }
}
```

#### I.2.3 Healthcare Prescription Authorization

```pseudocode
operation_types = {
  "controlled_substance_rx": "Authoritative",
  "patient_lookup": "NonAuthoritative"
}

policy_rules = {
  "controlled_substance_rx": {
    NORMAL: { action: "allow", constraints: [peer_notification] },
    STRESSED: { action: "allow", constraints: [supervisor_review_required] },
    DURESS: { action: "require", compensating: [in_person_verification, witness] },
    IMPAIRED: { action: "deny", reason: "cognitive_impairment_detected" },
    UNAVAILABLE: { action: "deny", reason: "attestation_required" }
  }
}
```

#### I.2.4 Enterprise Root Access

```pseudocode
operation_types = {
  "root_access": "Authoritative",
  "data_deletion": "Authoritative",
  "log_view": "NonAuthoritative"
}

policy_rules = {
  "data_deletion": {
    NORMAL: { action: "allow", constraints: [confirmation_required, logged] },
    STRESSED: { action: "allow", constraints: [second_admin_approval, delay_300s] },
    DURESS: { action: "trigger_silent_alarm", route: "security_team" },
    IMPAIRED: { action: "deny", reason: "operator_state", escalate: true },
    UNAVAILABLE: { action: "require", compensating: [break_glass_approval] }
  }
}
```

#### I.2.5 Industrial Safety Override

```pseudocode
operation_types = {
  "safety_system_override": "Authoritative",
  "parameter_view": "NonAuthoritative"
}

policy_rules = {
  "safety_system_override": {
    NORMAL: { action: "allow", constraints: [supervisor_present, logged] },
    STRESSED: { action: "allow", constraints: [safety_officer_approval, witnessed] },
    DURESS: { action: "deny", alert: [safety_team, management] },
    IMPAIRED: { action: "deny", lock: [privileged_controls] },
    UNAVAILABLE: { action: "require", compensating: [manual_key_authorization] }
  }
}
```

### I.3 Binding Field Examples

#### I.3.1 Session Binding

```pseudocode
# At authentication
session = create_session(user_credentials)
session_id = hash(session.auth_token || session.timestamp)

# When operation attempted
attestation = neural_lock.get_attestation()
attestation.session_id = session_id
operation.session_id = session_id

# Verification catches replay across sessions
if attestation.session_id != operation.session_id:
    deny("session_mismatch")
```

#### I.3.2 Intent Binding

```pseudocode
# Bitcoin spend
intent = {
  recipients: [(address_1, amount_1), (address_2, amount_2)],
  fee_rate: 10_sats_per_vbyte,
  change_address: address_3
}
intent_hash = canonical_hash(intent)

# Wire transfer
intent = {
  recipient_account: "123456789",
  amount: 50000_usd,
  currency: "USD",
  routing: "021000021"
}
intent_hash = canonical_hash(intent)

# Enterprise data deletion
intent = {
  resource_path: "/data/customer/records/2024",
  deletion_type: "permanent",
  reason: "retention_policy_expiry"
}
intent_hash = canonical_hash(intent)
```

### I.4 Graceful Degradation Examples

```pseudocode
function handle_unavailable(operation, policy):
    if operation.class == "NonAuthoritative":
        # Allow non-critical operations with warning
        warn_user("neural_lock_unavailable")
        return "ALLOW_WITH_WARNING"
    
    if operation.value < policy.low_value_threshold:
        # Allow low-value operations with enhanced logging
        log_audit("neural_lock_unavailable_low_value", operation)
        return "ALLOW_WITH_AUDIT"
    
    if policy.has_compensating_predicates:
        # Require additional authorization
        return "REQUIRE_GUARDIAN_APPROVAL"
    
    # High-value without compensating controls
    return "DENY"
```

## Annex J Quick Start Guide (Informative)

1. Collect a minimum 7-day baseline (Training Mode).
2. Configure policy (thresholds, required states, minimum confidence, freshness window).
3. Start in Advisory Mode and observe classifications and refusals.
4. Test with simulated stress and sensor loss scenarios.
5. Enable Enforcement Mode for low-value operations with caps and delays.
6. Gradually expand enforcement scope to higher-value operations.
7. Configure baseline reset governance and Break Glass before production use.
8. Run conformance tests (Annex E) in CI before releasing builds.

## Annex K Compliance Checklist (Informative)

* All artefacts use canonical CBOR encoding
* No floating-point values appear in any canonical artefact
* attestation_version is present and validated
* suite_profile is stable, versioned, and validated
* signal_sources are deterministically ordered
* DURESS is sticky and exit conditions are implemented
* UNAVAILABLE outcome is implemented and triggers Graceful Degradation
* Graceful Degradation does not silently downgrade security
* Conformance tests and Annex E vectors pass
* Privacy requirements met: no raw signals stored, transmitted, or logged
* Reason codes are local-only and not included in attestations
* Power management behaviour produces UNAVAILABLE during sleep and requires fresh signals after wake
* Liveness prompts are operation-class driven and do not leak coercion detection

---

## Changelog


### Version 1.1.0

**UNAVAILABLE Semantics Formalised**

* Introduced explicit `ClassifierOutcome` model.
* Defined strict rules for UNAVAILABLE attestation handling.
* Added normative Graceful Degradation integration with PQSEC ternary predicate model.

**Manual Duress Signal Added (4.12)**

* Optional, non-authoritative manual duress path.
* One-way DURESS transition semantics.
* Anti-coercion safety requirements and rotation rules.

**Evidence Producer Integrity Binding**

* Added `classifier_build_hash` requirement.
* Manifest option for multi-component classifiers.
* Explicit binding to CryptoSuiteProfile hash domain.

**Emission Discipline Strengthened**

* Operation-scoped attestation production only.
* Rate limiting requirements.
* External non-distinguishability requirements.
* ReceiptEnvelope transport alignment.

**Session & Intent Binding**

* Added optional `session_id` and `intent_hash` binding fields.
* Normative PQSEC binding validation requirements.

**Deployment Generalisation**

* Added 6A domain-agnostic integration section.
* Added Annex I generic gating interface examples.

**Training Mode Security Hardening**

* Added retraining frequency governance.
* Added baseline reset controls.
* Added operator variability accommodation rules.

**Conformance Surface Expanded**

* Expanded Annex E scenarios.
* Added Compliance Checklist (Annex K).
* Added liveness and power-management requirements.

---

## 16. Acknowledgements (Informative)

This specification acknowledges the foundational contributions of:

Ross Anderson, for practical security engineering and real-world coercion models.

Satoshi Nakamoto, for the Bitcoin protocol enabling policy-bound script execution.

The Bitcoin engineering community, for PSBT, Miniscript, and script-based policy composition.

Paul Kocher, Joshua Jaffe, and Benjamin Jun, for foundational work on timing and measurement attacks.

These contributions provide the threat models and primitives that Neural Lock composes into a non-authoritative human-state predicate.

---

If you find this work useful and wish to support continued development, donations are welcome:

**Bitcoin:**
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
