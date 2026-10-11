# Security Review: teslatlas-sdk-typescript

## Scope

Offline static Standard scan of frozen TypeScript SDK mirror at 1ac315f9.

- Scan mode: repository
- Target kind: git_revision
- Target ID: target_sha256_9d07658aebae7a1875efc877f4d9a5ed757f24635205ccdcad8db8200a7962fd
- Revision: 1ac315f975af87badd0b21ce93dd47ab014793b3
- Inventory strategy: repository
- Included paths: .
- Excluded paths: none
- Artifacts reviewed: AGENTS.md, .github/SECURITY.md, package.json, src/hub, src/http/bounded-json.ts, src/http/fetch-transport.ts, src/node.ts, src/browser.ts, scripts/browser-hub-runner.mjs, scripts/test-hub-browser-direct.mjs, scripts/test-hub-browser.mjs, selected generated validation and tests, parent master-plan migration entries

Limitations and exclusions:
- No build, tests, browser, live Hub, network or credentials.
- Mirror is frozen; authoritative retained Node code lives under Hub/sdk-typescript.
- Generated and legacy rich/browser/script corpus targeted rather than exhaustive.

### Scan Summary

| Field | Value |
| --- | --- |
| Scan outcome | completed |
| Reportable findings | 1 |
| Severity mix | low: 1 |
| Confidence mix | high: 1 |
| Coverage | partial |
| Validation mode | static_source_trace |

Canonical artifacts: `scan-manifest.json`, `findings.json`, and `coverage.json`. This report is a deterministic projection of those files.

## Threat Model

Frozen TypeScript SDK mirror includes retained Node Hub read/claim client and historically rich/browser/command paths slated for removal; authoritative copy lives in Hub workspace.

### Assets

- Hub pairing invitation and scoped bearer tokens
- Private vehicle/drive data and cursors
- Node client host/network trust store
- Browser helper token surface and local process

### Trust Boundaries

- Caller-provided Hub URL/invitation/credentials to Node TLS/fetch transport
- Remote Hub discovery, JSON and errors into client models/persistence
- Browser origin/page to helper or token-serving endpoints
- Repository generated contract/binding to runtime validation

### Attacker Capabilities

- Malicious or compromised configured Hub may return crafted JSON, redirects, oversized data or forged identity
- Local network actor may attempt TLS interception, redirect or proxy abuse
- Untrusted web origin may try to invoke browser helper if deployed
- Contributor may tamper generated/binding artifacts if accepted

### Security Objectives

- Never send pairing secret or bearer before authenticating intended Hub origin/pin
- Reject cross-origin redirects and insecure non-loopback HTTP
- Bound response parsing/resource work and validate identity/cursor ownership
- Keep browser helpers from exposing secrets to unrelated origins

### Assumptions

- Frozen mirror; retained client moves to authoritative Hub/sdk-typescript and rich/browser/commands paths are scheduled for deletion.
- Offline static read-only scan; no live Hub, browser, network, credentials or builds.
- No published v7 package and no browser client in v7 per repository guidance.

## Findings

| Finding | Severity | Confidence | Detailed write-up |
| --- | --- | --- | --- |
| [Legacy browser helpers expose a Hub bearer on unauthenticated loopback HTTP](#finding-1) | low | high | inline below |

### Confidence Scale

| Label | Meaning |
| --- | --- |
| high | Direct evidence supports the finding with no material unresolved blocker. |
| medium | Evidence supports a plausible issue, but material runtime or reachability proof remains. |
| low | Evidence is incomplete and the item is retained only for explicit follow-up. |

<a id="finding-1"></a>

### [1] Legacy browser helpers expose a Hub bearer on unauthenticated loopback HTTP

| Field | Value |
| --- | --- |
| Severity | low |
| Confidence | high |
| Confidence rationale | The credential read, config inclusion, 127.0.0.1 listener and unauthenticated response are explicit in both helper sources. |
| Category | credential_exposure |
| CWE | CWE-200, CWE-522 |
| Affected lines | scripts/browser-hub-runner.mjs:228, scripts/test-hub-browser-direct.mjs:119 |

#### Summary

Two browser acceptance helpers read an owner-only Hub credential file and embed the credential in a JSON config served by unauthenticated GET /config.json on 127.0.0.1. During a test run, another local OS user or process can fetch that route and obtain a device-scoped bearer despite not being able to read the source file. These helpers remain only in the frozen mirror; the authoritative Hub copy deleted them and v7 has no browser client.

#### Root Cause

Acceptance tooling republishes file-protected bearer material into a loopback endpoint without access control.

**file** — `scripts/browser-hub-runner.mjs:28-31`

Live credential enters the test helper from a private file.

```javascript
const credential = JSON.parse(await readFile(required("TESLATLAS_HUB_CREDENTIAL_FILE"), "utf8"));
```

**config** — `scripts/browser-hub-runner.mjs:69-75`

The config object retains the credential without redaction.

```javascript
const config = { endpoint: ..., hubId: ..., credential, driveLimit, expectedDriveCount: ... };
```

**listen** — `scripts/browser-hub-runner.mjs:165-168`

A loopback HTTP listener is reachable by other local processes/users.

```javascript
server.listen(port, "127.0.0.1", resolve)
```

**route** — `scripts/browser-hub-runner.mjs:226-231`

No authentication or caller identity check protects the credential-bearing response.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

**alternate** — `scripts/test-hub-browser-direct.mjs:114-122`

The second helper repeats the same disclosure after reading the owner-only credential.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

#### Validation

Traced both independent browser helpers from credential file read through config and unauthenticated loopback response; confirmed test script is exposed in package scripts.

Validation method: static_source_trace

**file** — `scripts/browser-hub-runner.mjs:28-31`

Live credential enters the test helper from a private file.

```javascript
const credential = JSON.parse(await readFile(required("TESLATLAS_HUB_CREDENTIAL_FILE"), "utf8"));
```

**config** — `scripts/browser-hub-runner.mjs:69-75`

The config object retains the credential without redaction.

```javascript
const config = { endpoint: ..., hubId: ..., credential, driveLimit, expectedDriveCount: ... };
```

**listen** — `scripts/browser-hub-runner.mjs:165-168`

A loopback HTTP listener is reachable by other local processes/users.

```javascript
server.listen(port, "127.0.0.1", resolve)
```

**route** — `scripts/browser-hub-runner.mjs:226-231`

No authentication or caller identity check protects the credential-bearing response.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

**alternate** — `scripts/test-hub-browser-direct.mjs:114-122`

The second helper repeats the same disclosure after reading the owner-only credential.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

Counterevidence and remaining uncertainty:
- Loopback bind limits remote reachability.
- Test lifecycle is transient, and CORS blocks ordinary cross-origin browser reads.
- Authoritative Hub copy has already removed these helpers per master plan.

Limitations:
- No live browser/helper process or credential used.

#### Dataflow

The canonical finding records the affected path at scripts/browser-hub-runner.mjs:228, scripts/test-hub-browser-direct.mjs:119, but no expanded source-to-sink narrative was recorded.

- **Source:** TESLATLAS_HUB_CREDENTIAL_FILE parsed credential

- **Sink:** JSON.stringify(config) HTTP response

- **Outcome:** Credential read by arbitrary local HTTP client

**file** — `scripts/browser-hub-runner.mjs:28-31`

Live credential enters the test helper from a private file.

```javascript
const credential = JSON.parse(await readFile(required("TESLATLAS_HUB_CREDENTIAL_FILE"), "utf8"));
```

**config** — `scripts/browser-hub-runner.mjs:69-75`

The config object retains the credential without redaction.

```javascript
const config = { endpoint: ..., hubId: ..., credential, driveLimit, expectedDriveCount: ... };
```

**listen** — `scripts/browser-hub-runner.mjs:165-168`

A loopback HTTP listener is reachable by other local processes/users.

```javascript
server.listen(port, "127.0.0.1", resolve)
```

**route** — `scripts/browser-hub-runner.mjs:226-231`

No authentication or caller identity check protects the credential-bearing response.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

**alternate** — `scripts/test-hub-browser-direct.mjs:114-122`

The second helper repeats the same disclosure after reading the owner-only credential.

```javascript
if (request.url === "/config.json") { response.setHeader("content-type", "application/json"); return response.end(JSON.stringify(config)); }
```

#### Reachability

The source file permissions do not restrict HTTP clients on loopback.

- **Attacker:** Another local OS user or process able to connect to loopback

- **Entry point:** GET http://127.0.0.1:\<helper-port\>/config.json during browser acceptance

- **Outcome:** Disclosure of device-scoped Hub bearer until expiry or revocation

Preconditions:
- A browser helper is running with a live Hub credential.
- The attacker can connect locally during its short lifetime.

Limitations:
- No public-network exposure or ordinary cross-origin browser JavaScript read is established.
- Helpers have been deleted in authoritative Hub/sdk-typescript; frozen mirror remains vulnerable.

#### Severity

**Low** — The barrier between an owner-only file and another local user is crossed, but the endpoint is loopback-only and transient developer/test tooling, not the retained v7 runtime.

Higher if this helper is run on a multi-user host with valuable live Hub credentials for long periods; removed when deprecated helpers are deleted from the mirror.

#### Remediation

Delete both retired browser helpers from this mirror when its archival policy permits. If test tooling is retained, do not publish bearer bytes over HTTP; inject them through an owner-only automation channel and ensure teardown.

Tests:
- With a helper running, an unauthenticated local GET /config.json must not return a credential.
- Verify the retained Node claim/client tests do not depend on either browser helper.

## Reviewed Surfaces

| Surface | Risk Area | Outcome | Notes |
| --- | --- | --- | --- |
| Rich client/events/commands | Legacy request and stream input | Needs follow-up | Targeted only; slated deletion. Ajv unique string checks are hash-like, not the Swift quadratic issue. |
| Generated binding and scripts | Contract/provenance/tool inputs | Needs follow-up | Relevant generated/binding scripts targeted, not exhaustive. |
| Node claim TLS and pinning | Invitation secret/credential transit | No issue found | Default Node claim verifies CA, hostname and leaf pin before secret write. |
| Retained Hub client and validation | Bearer/origin/cursor and JSON bounds | No issue found | Fixed routes, redirect error, origin/UUID admission and 1MiB bounded response reviewed. |
| Browser helpers and web trust | CORS/token exposure | Reported | Legacy helpers serve bearer in unauthenticated loopback config. |

## Open Questions And Follow Up

- Whether frozen mirror will be archived/deleted after authoritative Hub copy migration is owner-controlled.
- Full generated, historical rich/browser and script corpus, plus live browser/Hub behavior, not exhaustively assessed.
  - Follow-up prompt: Review deferred unit ts-residual-review and close its stated proof gap. Paths: src/generated, src/client, src/events, scripts.
