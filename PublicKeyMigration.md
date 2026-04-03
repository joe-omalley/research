# Public Key Verification Material Migration to IBM Security Verify SaaS

## Executive summary

This paper describes a safe approach for migrating and managing public verification material (public keys, certificates, JWKS entries) used by a phone‑authentication verification flow into IBM Security Verify SaaS, using only supported tenant APIs and documented integration surfaces.

The recommended model is a **hybrid pipeline**: perform a bulk onboarding of the existing, stable key set into Verify using the specific keystore or certificate administration APIs exposed by the tenant, then support **just‑in‑time (JIT)** onboarding for new keys via a small, hardened service that calls those same APIs. All configuration changes are performed through an API client that is granted minimal required access to the Verify tenant configuration surfaces.[cite:1][cite:2]

This document treats all concrete endpoints and payloads as **illustrative**. Exact URLs, resource names, and schemas must be taken from the Verify API documentation for the target tenant and validated in a staging environment before production use.[cite:1][cite:3]

## 1. Scope, assumptions, and constraints

### 1.1 Scope

This paper covers management of **verification material** used to validate signed artifacts produced by a phone‑authentication system, for example:

- Public keys used to validate JSON Web Tokens (JWTs) or signed payloads.
- X.509 certificates and chains that contain those public keys.
- JSON Web Key Sets (JWKS) that expose such keys via HTTP for OIDC/OAuth2‑based verification flows.[cite:4][cite:5]

Private keys, HSM material, and TLS server certificates used by Verify itself are out of scope.

### 1.2 Assumptions

- The Verify tenant exposes configuration and application management via REST APIs, secured by an API client and OAuth2 tokens.[cite:1][cite:2]
- The phone‑authentication verifier can be configured to trust public keys or certificates managed by the Verify tenant, or to consume JWKS exposed by Verify or by a separate endpoint under the same trust domain.[cite:4][cite:5]
- The current phone‑auth system has an exportable inventory of verification material (public keys, certificate chains, `kid` values, algorithms, and issuer metadata).

### 1.3 Constraints and safety requirements

- **No private key material** is migrated into Verify as part of this effort.
- Secrets such as API client credentials and any passphrases used for keystores must be stored in an enterprise secrets manager, not in scripts or source code.[cite:2]
- Any configuration artifact that embeds PEM or JWKS content must be protected by encryption‑at‑rest and strict RBAC.
- All changes are validated in a non‑production Verify tenant that is configured as close as possible to production before rollout.

## 2. Conceptual migration approaches

The migration problem can be phrased as: “How should the Verify tenant be made aware of the verification material needed for phone‑auth?” The answer depends on the format and lifecycle of that material.

### 2.1 Approaches overview

| Approach | Scale | Latency to use | Auditability | Primary use case |
|---------|-------|----------------|--------------|------------------|
| Bulk keystore or certificate collection | High | Single change event | High (single artifact under change control) | Initial migration of a large, mostly static key set |
| Batch certificate import via tenant APIs | Medium–High | Script‑driven | High (per‑cert records in tenant config) | Migration where verification material is natively X.509 |
| JWKS publication (Verify or external) | Any | Immediate once endpoint is live | Medium–High (changes via config or deployment) | OIDC/OAuth2 JWT verification, dynamic key rotation | 
| JIT per‑client onboarding (API‑driven) | Low–Medium per event | Immediate once API call succeeds | Medium (depends on service logging and approval model) | Onboarding of new clients or tenants with their own keys |

IBM documentation confirms JWKS as a standard integration pattern for JWT verification, and Verify supports OIDC/OAuth2 features that expose JWKS to relying parties.[cite:4][cite:5] Tenant configuration changes, including JWKS‑related settings, are performed through authenticated API clients over REST.[cite:1][cite:2]

### 2.2 Bulk vs JIT trade‑offs

- **Bulk/batch approaches**:
  - Pros: predictable change windows, fewer moving parts, easier to version and roll back, good fit for “catching up” with an existing estate.
  - Cons: do not on their own solve ongoing onboarding; require separate handling for keys introduced after the migration date.

- **JIT approaches**:
  - Pros: support continuous onboarding and rotation; new keys become usable shortly after they are registered via API.
  - Cons: introduce operational and security complexity (approval workflow, validation, config drift) and therefore must be mediated by a hardened service.

The recommended target architecture combines both: **bulk/batch for the baseline** plus a **governed JIT onboarding path** for incremental changes.

## 3. Target architecture and flows

### 3.1 High‑level components

The target state comprises the following components:

- **Verify tenant**: exposes admin APIs for configuration and application management, and runtime endpoints (OIDC discovery, JWKS, token endpoints) to upstream verifiers.[cite:1][cite:4][cite:5]
- **Phone‑auth backend**: continues to issue signed artifacts (tokens/assertions) using its own keys; only its public verification material is relevant here.
- **Verification material inventory**: a canonical database or manifest capturing for each key: `kid`, algorithm, key type, certificate chain (if any), issuer, validity, and owner system.
- **Migration and JIT onboarding service**: a small service that executes tenant configuration changes via the Verify APIs, performing validation and logging for every change.

### 3.2 Core flows

1. **Initial migration** (bulk):
   - Inventory and normalize the current verification material.
   - Map each artifact to the appropriate Verify configuration surface (keystore, certificate store, JWKS, or application‑level JWKS configuration).
   - Apply changes to a staging tenant via admin APIs.
   - Test end‑to‑end verification using both old and new key sources.
   - Apply the same changes to production during a controlled window, with a dual‑key acceptance period.

2. **Ongoing onboarding and rotation** (JIT through service):
   - Phone‑auth or client onboarding processes submit new verification material to the onboarding service (e.g., REST call with a JWK or certificate chain).
   - The service validates and persists the submission, then calls the relevant Verify API to register or update the key configuration.
   - The service records the change and returns a status to the caller.

3. **Reconciliation and governance**:
   - A scheduled task compares the canonical inventory to the Verify tenant configuration.
   - Differences (missing, extra, or mismatched keys) are flagged and remediated in a controlled process.

## 4. Technical details and invocation patterns (illustrative)

> **Important:** All endpoints and payloads in this section are conceptual. They are provided to illustrate patterns only. Concrete shapes must be obtained from the Verify tenant API documentation and confirmed in staging before any production use.[cite:1][cite:3]

### 4.1 Authentication and authorization

Pattern:

- Create an **API client** in the Verify tenant with the minimum required administrative scopes to manage configuration surfaces involved in key exposure.[cite:2]
- Obtain OAuth2 tokens (typically using the client credentials grant) from the tenant’s authorization server.

Illustrative token request:

```http
POST https://<tenant-domain>/v1.0/endpoint/default/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>
```

The actual token endpoint URL, client authentication method (plain client secret, `private_key_jwt`, or mTLS), and scopes must follow the tenant’s configuration and documentation.[cite:4][cite:5]

### 4.2 Bulk/batch onboarding of existing material

Bulk onboarding can take one of two broad shapes, depending on what the Verify tenant actually exposes:

1. **Keystore‑like resource**: a single logical container that holds multiple public keys or certificates.
2. **Per‑certificate or per‑key resources**: a collection of certificate objects or key records in tenant configuration.

#### 4.2.1 Keystore‑like resource (conceptual)

If the tenant exposes a keystore resource that accepts a PKCS#12 (.p12) bundle of certificates, a typical pattern is:

1. Build a PKCS#12 file that contains the X.509 certificates corresponding to the phone‑auth public keys.
2. Upload the keystore via an admin API under the tenant configuration namespace.
3. Patch a higher‑level configuration object (for example, a "phoneAuth" section) to reference the new keystore.

Illustrative upload pattern:

```http
POST https://<tenant-domain>/v1.0/tenants/{tenantId}/configuration/keystores
Authorization: Bearer <access_token>
Content-Type: application/octet-stream
X-Keystore-Password: <p12_password>

<binary pkcs12 body>
```

Illustrative activation pattern:

```http
PATCH https://<tenant-domain>/v1.0/tenants/{tenantId}/configuration
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "phoneAuth": {
    "verificationKeystoreId": "<keystore_id>"
  }
}
```

These shapes are not authoritative and must be replaced with the actual resource definitions from the Verify API documentation.

#### 4.2.2 Batch certificate import (conceptual)

Where the tenant exposes a certificate management API, an alternative is to import each certificate as its own resource.

Illustrative pattern:

```http
POST https://<tenant-domain>/v1.0/tenants/{tenantId}/configuration/certificates
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "name": "phone-key-001",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIB...==\n-----END CERTIFICATE-----"
}
```

The onboarding service must map each certificate in the canonical inventory to a corresponding tenant record, recording the relationship in the inventory for later reconciliation.

### 4.3 JWKS‑based exposure

For flows that use JWTs and OIDC/OAuth2, the preferred pattern is often to rely on JWKS, either hosted by Verify or by a separate endpoint under the organization’s control.[cite:4][cite:5]

There are two sub‑patterns:

1. **Tenant or provider JWKS**: Verify hosts a JWKS endpoint as part of its OIDC provider configuration.
2. **Application‑level JWKS**: individual applications in Verify are configured with embedded JWKs or with references to external JWKS URLs.

In both cases, the onboarding service maintains the canonical JWK representation (including `kid`, `kty`, `alg`, `n`, `e` for RSA keys, etc.) and updates either the Verify configuration or an external JWKS endpoint accordingly.

Illustrative application configuration patch:

```http
PATCH https://<tenant-domain>/v1.0/applications/{appId}
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "jwks": {
    "keys": [
      { "kty": "RSA", "kid": "abc", "alg": "RS256", "n": "...", "e": "AQAB" }
    ]
  }
}
```

Actual schemas and field names must match the Verify application configuration API.

### 4.4 What not to do

- Do not fabricate self‑signed certificates around arbitrary public keys just to fit a keystore API. Only use real certificate material that matches the actual keys issued and trusted by the system.
- Do not rely on undocumented APIs discovered solely by inspecting browser traffic without validating them in the product documentation or support channels.
- Do not embed long‑lived secrets or passphrases directly in scripts or YAML; use a secrets manager and runtime injection.[cite:2]

## 5. Recommended hybrid migration pipeline

### 5.1 Phase A — Preparation and inventory

1. **Inventory current verification material**:
   - Extract all relevant public keys, certificates, and JWKS entries from the existing phone‑auth and verification infrastructure.
   - Record, at minimum, `kid`, algorithm, key type, certificate subject/issuer (if applicable), validity dates, and owning system.

2. **Classify by format and target surface**:
   - Map each artifact to one of: certificate store, keystore, tenant JWKS, application JWKS, or external JWKS.
   - Decide, per artifact class, which Verify configuration surface will host or reference it.

3. **Normalize and validate**:
   - Ensure all certificates are in a consistent PEM format and validate their chains.
   - For JWKs, confirm they match the expected algorithms and key sizes for the phone‑auth verifier.
   - Define canonical `kid` values and resolve duplicates.

4. **Design the approval and change workflow**:
   - Establish who can approve new keys, who operates the onboarding service, and what evidence is required for each change.

### 5.2 Phase B — Bulk/batch baseline into staging

1. Build the initial configuration artifacts (keystore bundle, certificate collection, and/or JWKS entries) based on the normalized inventory.
2. Use the admin APIs with a staging tenant to import the artifacts.
3. Configure the staging phone‑auth verification path to:
   - Continue trusting the existing key source, and
   - Start trusting the new Verify‑backed source (dual‑key acceptance window).
4. Run end‑to‑end tests:
   - Verify JWT/signature validation succeeds for all expected `kid` values against the Verify‑backed configuration.
   - Confirm error behavior for unknown or expired keys.

5. Iterate until the staging configuration is stable and all test cases pass.

### 5.3 Phase C — Production rollout with dual‑key window

1. Apply the validated configuration changes to the production tenant via the same automated mechanism used for staging.
2. Enable a **dual‑key acceptance** period:
   - The verifier continues to accept artifacts validated by the legacy key source.
   - The verifier also accepts artifacts validated by the new Verify‑managed key source.

3. Monitor during the dual‑key window:
   - Error rates and key resolution failures.
   - Any unexpected `kid` or issuer values.

4. Once confidence is achieved, decommission the old key source and remove its configuration, retaining snapshots for rollback.

### 5.4 Phase D — JIT onboarding service

Deploy a small, hardened service that acts as the only path for dynamically onboarding or rotating keys.

Key responsibilities:

- **Ingress**:
   - Accepts requests from approved callers (e.g., phone‑auth backend, client onboarding process) over mTLS or authenticated REST.
   - Accepts only well‑formed payloads (JWK, JWKS reference, or certificate chain) with clear metadata (owner, environment, expected usage).

- **Validation**:
   - Validates algorithm, key size, `kid` uniqueness, and certificate chain integrity.
   - Checks for conflicts with existing inventory entries.

- **Tenant update**:
   - Calls the appropriate Verify admin API (keystore, certificate, or JWKS configuration) to register the new key.
   - Updates the canonical inventory with the new record and its tenant identifier.

- **Logging and audit**:
   - Records who requested the change, what material was added or changed (by fingerprint), and the result of the tenant update.

### 5.5 Phase E — Reconciliation and rotation

1. **Daily or frequent reconciliation**:
   - Export the relevant configuration from the Verify tenant via admin APIs.
   - Compare against the canonical inventory to detect drift (keys present only in tenant, only in inventory, or with mismatched metadata).

2. **Planned rotation**:
   - For each rotation event, add the new key or certificate, update the verifier and/or JWKS to prefer the new `kid`, then retire the old key after a safe overlap window.

3. **Rollback model**:
   - Maintain versioned snapshots of:
      - The canonical inventory.
      - The applied Verify configuration (e.g., JSON representations of keystores/certificates/JWKS where exposed by APIs).
   - In a controlled rollback, apply the previous configuration snapshot to the tenant and adjust the verifier configuration accordingly.

## 6. Testing, validation, and operational guardrails

### 6.1 Testing checklist

- Confirm API client authentication and authorization to the relevant Verify admin APIs.[cite:2]
- Verify that keystore or certificate imports succeed and are reflected in tenant configuration exports.
- Confirm JWKS endpoints (tenant or application) expose the expected keys, `kid` values, and algorithms used by the phone‑auth verifier.[cite:4][cite:5]
- Run signed phone‑auth payloads through both the legacy and Verify‑backed verification paths during the dual‑key window.
- Validate that error paths (unknown `kid`, invalid algorithm, expired certificate) behave as designed.

### 6.2 Operational guardrails

- **Least privilege and access control**: the admin API client used by the onboarding service should have only the scopes required for key‑related configuration changes.[cite:2]
- **Idempotency**: scripts and services should detect existing key material (by fingerprint or `kid`) and update in place rather than create duplicates.
- **Inventory as source of truth**: treat the canonical inventory as the single source of truth for verification material; the tenant configuration should be a projection of that inventory.
- **Secrets hygiene**: store API client credentials, keystore passwords, and any other secrets in a managed secrets system and inject them at runtime.
- **Logging and monitoring**: centralize logs from the onboarding service and from Verify audit trails to detect unauthorized or unexpected key changes.[cite:1]

## 7. Risks and mitigations

### 7.1 Undocumented or unstable endpoints

- **Risk**: using endpoints or resources that are not part of the public Verify documentation may lead to breaking changes or unsupported configurations.
- **Mitigation**:
   - Limit use to documented APIs and patterns.[cite:1][cite:3]
   - Validate behavior in staging and consult IBM support where necessary.

### 7.2 Format mismatches and verification failures

- **Risk**: keys in the tenant configuration do not match the actual signing keys used by the phone‑auth system (wrong algorithm, wrong key size, wrong certificate chain), causing verification failures.
- **Mitigation**:
   - Strict validation in the onboarding service.
   - Cross‑checking sample signed payloads against both the inventory and the tenant configuration.

### 7.3 Configuration drift (especially under JIT)

- **Risk**: keys are added or changed via ad‑hoc mechanisms (console, one‑off scripts) and not reflected in the canonical inventory.
- **Mitigation**:
   - Mandate that all changes go through the onboarding service and inventory.
   - Run daily reconciliation and investigate any discrepancies.

### 7.4 Security exposure of embedded material

- **Risk**: embedding PEMs or JWKS in configuration files or scripts creates additional attack surface.
- **Mitigation**:
   - Minimize embedding; favor references to managed keystores or JWKS endpoints.
   - Use encryption‑at‑rest and strict RBAC for any remaining embedded material.[cite:1]

## 8. Next deliverables

The following follow‑on artifacts can be produced from this design:

- A **migration runbook** with tenant‑specific endpoints, exact commands, and rollback procedures.
- A **test plan** with detailed test vectors for JWT and certificate validation, including negative cases.
- A **detailed specification for the JIT onboarding service**, including API contracts, validation rules, and example implementation snippets.
```
