# PartnerIntegrationFacadeAPI (v1.0) - Documentation

## 1. Requirement Summary

This API proxy serves as an enterprise facade for partner ingress. It enables conditional backend routing based on partner tier, enforces OAuth2 access, applies traffic spike arrest, and mediates certain headers for security and observability.

**Key Requirements**:
- Partner ingress via `/v1/partner-ingress`
- Conditional routing for "Premium" partners
- OAuth2 access enforcement on all requests
- Spike arrest rate limiting (15 requests/sec)
- Header mediation: remove sensitive incoming headers, add timestamp header

---

## 2. Proxy Details

| Property           | Value                                                       |
|--------------------|-------------------------------------------------------------|
| **Proxy Name**     | PartnerIntegrationFacadeAPI                                 |
| **Version**        | 1.0                                                         |
| **Base Path**      | `/v1/partner-ingress`                                       |
| **Description**    | Enterprise abstraction facade for partner ingress with conditional routing based on partner tier, OAuth2 access enforcement, spike arrest, and mediation. |

---

## 3. Routing Rules

| Rule Name          | Condition                                           | Target Endpoint                |
|--------------------|----------------------------------------------------|-------------------------------|
| premium-partner    | `request.header.X-Partner-Tier = "Premium"`        | Target-Enterprise-Cluster     |
| default            | (always matches if above fails)                    | Target-Standard-Cluster       |

---

## 4. Target Endpoints

### Target-Enterprise-Cluster
- **URL:** `https://premium-api.internal.net/v1/data`
- **Used for:** Requests where `X-Partner-Tier: Premium`

### Target-Standard-Cluster
- **URL:** `https://standard-api.internal.net/v1/data`
- **Used for:** All other requests

---

## 5. Policies

### 5.1 OAuthV2 Policy (`oauth-verify`)
- **Type:** OAuth2 Access Token Verification
- **Operation:** `VerifyAccessToken`
- **Token Location:** Request header `Authorization`

### 5.2 AssignMessage Policy (`assign-partner-header-mediation`)
- **Removes Headers:** `X-Internal-Token`, `Cookie`
- **Adds Header:** `X-Gateway-Timestamp: {system.timestamp}`
- **AssignTo:** Current request, does not create new message

### 5.3 SpikeArrest Policy (`spike-arrest`)
- **Rate:** 15 requests per second (15ps)
- **Applied on:** PreFlow (all requests)

---

## 6. Generated Files

```
apiproxy/
├── PartnerIntegrationFacadeAPI.xml                        (Proxy Descriptor)
├── proxies/
│   └── default.xml                                       (ProxyEndpoint configuration)
├── targets/
│   ├── Target-Enterprise-Cluster.xml                     (TargetEndpoint for premium)
│   └── Target-Standard-Cluster.xml                       (TargetEndpoint for standard)
└── policies/
    ├── oauth-verify.xml                                  (OAuth2 policy)
    ├── spike-arrest.xml                                  (Spike Arrest policy)
    └── assign-partner-header-mediation.xml               (AssignMessage policy)
```

---

## 7. Security Design

- **OAuth2 Verification:** Strict verification of bearer tokens on every request.
- **Header Scrubbing:** Sensitive headers (`X-Internal-Token`, `Cookie`) removed before forwarding upstream.
- **Replay/Trace Prevention:** Adds `X-Gateway-Timestamp` (injected with gateway's system time) to every request for downstream auditing/correlation.
- **Rate Limiting:** Upstream protected by 15 requests-per-second cap to prevent DoS/spikes.

---

## 8. Request Flow

1. **Client sends request** to `/v1/partner-ingress` with OAuth2 token and headers.
2. **PreFlow policies** enforce:
   - **OAuth2 Token Verification**
   - **Header Mediation** (removed: `X-Internal-Token`, `Cookie`; added: `X-Gateway-Timestamp`)
   - **Spike Arrest** (rate limit)
3. **Routing Decision**:
   - If `X-Partner-Tier: Premium` → Target-Enterprise-Cluster
   - Otherwise → Target-Standard-Cluster
4. **Request forwarded** to the selected backend URL.
5. **Response flows** untouched back to the client.

---

## 9. Assumptions

- All partners must present a valid OAuth2 access token.
- `X-Partner-Tier` header is trustworthy and not user-spoofable, or is at least validated externally.
- Backend target URLs (`premium-api.internal.net`, `standard-api.internal.net`) are routable from Apigee.
- No message transformation or content filtering is required except for header mediation.
- 15ps spike arrest is sufficient for all partners at this ingress point.

---

## 10. Clarifications Required

- How is the `X-Partner-Tier` header populated? Is additional validation required to prevent spoofing?
- Is response header mediation required (e.g., should certain internal headers also be scrubbed before responses are sent back to clients)?
- Should spike arrest be applied globally for all partners, or by client/partner key?
- Are there any requirements on HTTP method restrictions (GET/POST/etc.), or is all traffic allowed?
- Should error messages from failed OAuth/token checks be customized?
- Any requirements around detailed logging/auditing for regulatory purposes?

---

## 11. Deployment Structure

- **Proxy as Single Bundle:** All policies and endpoints (proxy, targets, policies) defined as a single Apigee API proxy.
- **Environments:** Deployable to Apigee X environments (e.g., `dev`, `test`, `prod`) as per standard Apigee CI/CD.
- **Proxy Endpoint:** Exposed on `/v1/partner-ingress` (as set by `BasePath`).
- **No Custom JavaScript or Shared Flows** specified in current design.

---

## 12. Testing Recommendations

**Functional Tests**
- ✔️ Valid OAuth token, "Premium" tier → routes to `https://premium-api.internal.net/v1/data`
- ✔️ Valid OAuth token, non-premium tier → routes to `https://standard-api.internal.net/v1/data`
- ✔️ Invalid/missing OAuth token → access denied (401/403)
- ✔️ Header mediation removes `X-Internal-Token` and `Cookie` before forwarding; adds `X-Gateway-Timestamp`
- ✔️ Spike arrest enforced (15 requests in 1 sec allowed, 16th+ are rejected)

**Negative Tests**
- ❌ Over-rate requests result in 429 error
- ❌ Malformed/expired token is rejected

**Security Tests**
- 🔒 Attempt spoofing `X-Partner-Tier`, ensure policy is correct for intended scenario
- 🔒 Inspect backend to confirm no leaked headers

**Performance Tests**
- 🔬 Validate performance does not degrade under expected concurrency

**Integration Tests**
- ↔️ Ensure correct behavior with actual backend clusters

**Edge/Regression Cases**
- 🧪 Requests with missing, empty, or malformed headers

---

*End of documentation.*