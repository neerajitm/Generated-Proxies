# TestSecureProxy - API Proxy Documentation

## 1. Requirement Summary

- **Purpose:** Expose a secure API proxy that validates API keys and enforces spike arrest (5 requests/second) for the backend service `https://mocktarget.apigee.net`.
- **Security:** Protect API access using API key verification.
- **Traffic Control:** Prevent abuse with spike arrest policy.

---

## 2. Proxy Details

- **Proxy Name:** TestSecureProxy
- **Version:** 1.0.0
- **Base Path:** `/v1/secure-api`
- **Description:** API Proxy enforcing API key security and spike arrest (5 requests/sec) to mocktarget.apigee.net

---

## 3. Routing Rules

| Route Rule Name | Condition | Target Endpoint |
|:---------------:|:---------:|:--------------:|
| default         | (None)    | default        |

- All incoming requests matching `/v1/secure-api` are routed to the `default` target endpoint.

---

## 4. Target Endpoints

- **Name:** `default`
- **Target URL:** `https://mocktarget.apigee.net`

Target endpoint receives traffic after all proxy-side policies (API key check, spike arrest) succeed.

---

## 5. Policies

### **1. VerifyAPIKey (`verify-apikey`)**
- **Type:** VerifyAPIKey
- **Configuration:** Checks for an API key in the query parameter `apikey` (`request.queryparam.apikey`).
- **Enforced On:** All incoming requests in the PreFlow (Proxy Endpoint).

### **2. SpikeArrest (`spike-arrest`)**
- **Type:** SpikeArrest
- **Configuration:** Limits traffic to a maximum of 5 requests per second (`5ps`).
- **Enforced On:** All incoming requests in the PreFlow (Proxy Endpoint).

---

## 6. Generated Files

| File Path                                   | File Type      | Description                                |
|:--------------------------------------------|:---------------|:-------------------------------------------|
| apiproxy/TestSecureProxy.xml                | API Proxy      | API proxy main descriptor                  |
| apiproxy/proxies/default.xml                | Proxy Endpoint | Proxy endpoint configuration (`default`)    |
| apiproxy/targets/default.xml                | Target Endpoint| Target endpoint configuration (`default`)   |
| apiproxy/policies/verify-apikey.xml         | Policy         | API key verification policy                 |
| apiproxy/policies/spike-arrest.xml          | Policy         | Spike arrest (rate limiting) policy         |

---

## 7. Security Design

- **API Key Required:** 
  - All requests must provide a valid API key as the query parameter `apikey`.
  - Requests without a valid API key are rejected before hitting the backend.
- **Rate Limiting:**
  - Spike Arrest policy caps usage to 5 requests per second to prevent API abuse.
- **Backend Shielding:**
  - Backend system (`mocktarget.apigee.net`) is protected against unauthorized and excessive requests.

---

## 8. Request Flow

```mermaid
graph TD
    Client -->|HTTPS Request| ApigeeProxy
    ApigeeProxy -->|PreFlow: VerifyAPIKey| VerifyAPIKeyPolicy
    VerifyAPIKeyPolicy -- valid key -->|PreFlow: SpikeArrest| SpikeArrestPolicy
    SpikeArrestPolicy -- passed -->|Route| TargetEndpoint
    TargetEndpoint -->|Proxy| mocktarget.apigee.net
```

1. **Incoming request** to `/v1/secure-api` is received.
2. **PreFlow** applies the following policies in order:
    - `verify-apikey`: API key in `request.queryparam.apikey` is validated.
    - `spike-arrest`: Rate limit is checked (5 requests/sec per key).
3. If both policies pass, request is routed to `https://mocktarget.apigee.net`.
4. **Response** from the target is returned unchanged.

---

## 9. Assumptions

- API consumers are provisioned API keys in advance.
- No additional authentication or authorization policies are required.
- All requests should include the `apikey` query parameter.
- No transformation, logging, or response policies are configured (beyond defaults).
- Mocktarget is a suitable backend for test/demo purposes.

---

## 10. Clarifications Required

- Should API key also be accepted via headers or POST body (currently only query parameter supported)?
- Should the rate limit be applied globally or per API key (currently applies per proxy instance, default behavior is typically per key/client)?
- Are there any specific error message/response format requirements for failed API key or rate limit errors?
- Any plans to enhance security (e.g., OAuth, mTLS) or add threat protection (e.g., JSON Threat Protection, XSS/SQLi filters)?
- Should any CORS configuration be applied for frontend clients?
- Any environment-specific endpoints or mocktarget variants required?

---

## 11. Deployment Structure

```plaintext
apiproxy/
│
├── TestSecureProxy.xml
├── proxies/
│   └── default.xml
├── targets/
│   └── default.xml
└── policies/
    ├── verify-apikey.xml
    └── spike-arrest.xml
```

- Eclipse, Edge UI, or Apigee CLI can be used to deploy the structured bundle.
- On deployment, the proxy will expose `/v1/secure-api` for all traffic.

---

## 12. Testing Recommendations

1. **API Key Required Test**
   - Request WITHOUT `apikey` parameter → Expect HTTP 401/403 Unauthorized.
2. **Valid/Invalid API Key Test**
   - Request with an INVALID `apikey` → Expect HTTP 401/403 Unauthorized.
   - Request with a VALID `apikey` → Expect HTTP 200 and valid backend response.
3. **Spike Arrest Test**
   - Issue >5 requests/second using a VALID API key → Expect HTTP 429 Too Many Requests after threshold.
   - Issue ≤5 requests/second → All requests should succeed.
4. **Backend Connectivity Test**
   - Confirm backend responses pass through unchanged.
5. **Error Handling**
   - Confirm error details for both invalid API key and rate limit breach are clear and informative.
6. **Edge Cases**
   - Test with non-GET methods, missing/extra query parameters.
7. **Security Scan**
   - Verify no information disclosure in error messages.
8. **Load Test**
   - Simulate concurrent users with unique/duplicate API keys.

---

**End of documentation for TestSecureProxy**