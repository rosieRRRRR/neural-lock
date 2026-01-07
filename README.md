# Neural Lock — Human-State-Aware Policy Gate for Bitcoin Custody

**An Open Standard for Coercion-Resilient, Policy-Bound Bitcoin Operations**

* **Specification Version:** 1.0.0
* **Status:** Implementation Ready (Domain Evaluation Requested)
* **Date:** 2026
* **Author:** rosiea
* **Contact:** PQRosie@proton.me
* **Licence:** Apache License 2.0 — Copyright 2025 rosiea

---

## Abstract

Neural Lock is a non-authoritative policy evidence mechanism that conditions high-risk operations on locally-classified human cognitive and physiological state signals. By detecting risk-correlated states consistent with duress, coercion, impairment, or cognitive overload, Neural Lock reduces the success rates of coercion-driven compromise without replacing cryptographic authority. This specification defines the architecture, integration patterns, and security properties necessary for implementation in Bitcoin custody systems, for both self-custody and institutional custody, with particular focus on reducing the success rates of physical coercion attacks (the "$5 wrench attack") while preserving user sovereignty and privacy.

Key Innovation: Neural Lock adds an operator state predicate as a fourth dimension of Bitcoin security, complementing cryptographic keys, temporal controls (timelocks), and distributed authority (multisig).

---

## Index

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Architecture](#3-architecture)
4. [Signal Sources and State Classification](#4-signal-sources-and-state-classification)
5. [Policy Integration](#5-policy-integration)
6. [Bitcoin Integration Patterns](#6-bitcoin-integration-patterns)
7. [Security Properties](#7-security-properties)
8. [Privacy Protections](#8-privacy-protections)
9. [Threat Model](#9-threat-model)
10. [Integration with PQ Ecosystem](#10-integration-with-pq-ecosystem)
11. [Implementation Guidance](#11-implementation-guidance)
12. [Comparison to Existing Solutions](#12-comparison-to-existing-solutions)
13. [Future Work](#13-future-work)
14. [References](#14-references)
15. [Annexes](#15-annexes)
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
5. Revocable: users can disable Neural Lock or downgrade to advisory-only mode
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
* PQHD: Post-Quantum Hierarchical Deterministic Wallet
* PQEH: Post-Quantum Execution Hardening
* Multi-Predicate Custody: authorisation requiring multiple conditions
* ZET/ZEB: Zero-Exposure Transaction and Broadcast
---

## 2A. Explicit Dependencies

| Specification | Minimum Version | Purpose |
|---------------|-----------------|---------|
| PQSEC | ≥ 2.0.1 | Enforcement of operator_state_ok predicate |
| PQSF | ≥ 2.0.2 | Canonical encoding for NeuralAttestation |
| PQHD | ≥ 1.1.0 | Custody integration and predicate composition |
| Epoch Clock | ≥ 2.1.1 | Freshness binding for attestations |

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
* confidence_value in [0, 1000]

Confidence semantics:

* confidence represents signal coverage and freshness only
* confidence MUST NOT be interpreted as correctness or likelihood of coercion
* implementations MUST NOT expose "coercion probability" UI

### 4.6 State Persistence and Debounce

* minimum state duration: 5 seconds (debouncing)
* state transitions require sustained signal change
* DURESS is sticky as defined in Section 7.3

### 4.7 Sensor Failure Handling

Missing signals:

* if primary signals unavailable: reduce confidence, bias toward STRESSED where supported by remaining valid signals
* if signals are insufficient to classify: outcome MUST be UNAVAILABLE

Contradictory signals:

* weighted voting among enabled and valid signals
* if confidence below policy minimum: escalate to DURESS where classification is available

### 4.8 Sensor Spoofing and Liveness

Spoofing indicators (non-exhaustive):

* zero variance over a defined window
* abrupt reappearance after prolonged absence
* physiological vs behavioural mismatch

Spoofing classification:

1. classifier outcome MUST be AVAILABLE with NeuralState = DURESS
2. confidence MUST be set to minimum representable value
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

## 4.11 UNAVAILABLE Classification Semantics (Normative)

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

1. MUST remain local to the device,
2. MUST NOT include raw biometric or behavioural signal data, and
3. MUST NOT be exported or treated as enforcement artefacts.

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

    // Profile indirection (no hard-coded algorithms)
    pub suite_profile: String,
    pub signature: Vec<u8>,
}
```

### 5.2 Tick Units Requirement

freshness_window_ticks is expressed in the same units as the Epoch Clock tick. Implementations MUST NOT treat tick values as seconds unless the Epoch Clock profile defines that mapping.

### 5.3 suite_profile Registry Requirements (Minimal)

suite_profile MUST be a stable identifier for the cryptographic suite used to sign NeuralAttestation.

Requirements:

1. suite_profile MUST be stable across implementations within a deployment
2. suite_profile identifiers MUST be versioned
3. suite_profile MUST determine at minimum: signature algorithm family and parameters, public key format and verification rules, and domain separation rules if any
4. unknown suite_profile values MUST cause rejection

### 5.4 device_id Lifecycle and Privacy Requirements

device_id identifies the attestation-producing device within a deployment.

Requirements:

1. device_id MUST be stable for the duration of an enrollment
2. device_id SHOULD be derived to avoid global cross-deployment correlation
3. implementations SHOULD support device_id rotation by re-enrollment
4. device_id MUST NOT embed plaintext hardware serial numbers or globally unique identifiers

### 5.5 SignalSource Schema and Ordering (Normative)

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

### 5.6 Signature Input

The signature MUST be computed over the canonical CBOR encoding of the NeuralAttestation with signature omitted.

attestation_version MUST be included in the signed payload.

### 5.7 Policy Evaluation Order

Policy evaluation order for operations that require Neural Lock:

1. kill switch
2. classifier outcome evaluation
3. consent existence (ConsentProof)
4. policy caps and constraints
5. custody quorum and role constraints
6. signature validation and execution

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

### 6.4 Bitcoin Primitives Used

* CLTV and CSV
* standard P2WPKH, P2WSH, P2TR
* BIP32 derivation paths
* PSBT v2

### 6.5 Execution Revelation Gating (ZET and PQEH)

When used with ZET or PQEH execution patterns:

1. Neural Lock evaluation MUST complete before execution revelation
2. S1 revelation MUST NOT occur unless operator_state_ok is OK
3. if state enters DURESS or IMPAIRED after intent formation, execution MUST abort and S1 MUST NOT be revealed
4. UNAVAILABLE outcome MUST follow Section 11.8

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
| Full device compromise        | classifier runs on device   | PQVL plus secure boot plus isolation            |
| Correlated guardian coercion  | guardians coerced together  | quorum diversity, independent guardians, delays |
| Key extraction                | not a key-protection system | hardware isolation plus PQHD quorum             |
| Post-broadcast quantum attack | Bitcoin limitation          | PQEH plus ZET plus ZEB discipline              |

### 9.3 Honest Limitations

Neural Lock does not prove coercion. It reduces risk by detecting anomalous operator states correlated with elevated attack likelihood and applying fail-safe policy constraints.

---

## 10. Integration with PQSF Ecosystem

Neural Lock contributes exactly one predicate:

```
operator_state_ok
```

operator_state_ok evaluation:

* OK: NeuralAttestation is present and all required validation steps succeed for the operation class policy
* NOT_OK: NeuralAttestation is present but fails one or more required validation steps
* UNAVAILABLE: no NeuralAttestation is present because classifier outcome is UNAVAILABLE

Consuming enforcement systems MUST treat NOT_OK as predicate failure. UNAVAILABLE MUST be handled via Section 11.8 (Graceful Degradation).

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
3. bind the operation attempt deterministically to the NeuralAttestation via intent_hash (or operation_hash), and session binding where applicable
4. treat missing, stale, invalid, or unsupported NeuralAttestation as NOT_OK or UNAVAILABLE, and handle UNAVAILABLE via compensating controls or refusal

---

## 11. Implementation Guidance

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

Implementations claiming conformance MUST pass Annex E test vectors.

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

Recommended baseline survivability policy (informative):

* under UNAVAILABLE, allow only constrained, non-catastrophic operations (for example decoy access, strict caps, enforced delays, and guardian notification), and require guardians for high-value actions.

---

## 12. Comparison to Existing Solutions

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
* PQVL
* Epoch Clock
* ZET
* ZEB
* PQEH
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
        return { outcome: "AVAILABLE", state: DURESS, conf_value: 0, conf_scale: CONF_SCALE, reason: "spoofing_detected" }

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

## Annex B Spoofing and Liveness Detection Pseudocode

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

## Annex C NeuralAttestation Construction and Signing

```pseudocode
function build_attestation(classifier_result, policy, device, current_tick):
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
        suite_profile: policy.suite_profile
    }

    bytes = canonical_cbor_encode(att)
    sig = sign_with_profile(policy.suite_profile, device.signing_key, bytes)
    att.signature = sig
    return att
```

## Annex D PSBT Integration and Proprietary Field Versioning

```pseudocode
// Key: 0xFC | "neural_lock" | 0x01
function attach_neural_lock_to_psbt(psbt, attestation):
    key = proprietary_key(prefix=0xFC, ns="neural_lock", ver=0x01)
    psbt.proprietary[key] = canonical_cbor_encode(attestation)
    return psbt
```

## Annex E Conformance Test Vectors (Normative)

This annex defines test vectors as structured cases. Implementations MAY add additional vectors but MUST include these and MUST produce the expected outcomes.

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

* Expected: classifier outcome AVAILABLE with state DURESS and conf_value = 0

E.8 Sensor Loss Insufficient Signals

* Expected: classifier outcome UNAVAILABLE; no NeuralAttestation produced; operation handled via Section 11.8

E.9 Privacy Test No Raw Signal Leakage

* Expected: no raw signals stored, transmitted, or logged

## Annex F Decoy Wallet Derivation and Constraints

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

```pseudocode
OperationIntent = {
  operation_type: string,
  operation_class: "Authoritative" | "NonAuthoritative",
  intent_hash: bytes,
  session_id: string | null,
  exporter_hash: bytes | null,
  issued_tick: uint64,
  expiry_tick: uint64
}

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

    return "OK"
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

## 16. Acknowledgements (Informative)

This specification acknowledges the foundational contributions of:

Ross Anderson, for practical security engineering and real-world coercion models.

Satoshi Nakamoto, for the Bitcoin protocol enabling policy-bound script execution.

The Bitcoin engineering community, for PSBT, Miniscript, and script-based policy composition.

Paul Kocher, Joshua Jaffe, and Benjamin Jun, for foundational work on timing and measurement attacks.

These contributions provide the threat models and primitives that Neural Lock composes into a non-authoritative human-state predicate.

If you find this work useful and want to support it:

bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw

End of Specification
