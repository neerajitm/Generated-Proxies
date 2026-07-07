```markdown
# SimpleSecureProxy — Apigee X Proxy Documentation

## 1. Requirement Summary

**Objective:**  
Deploy a secure API proxy in Apigee X with:
- API Key authentication for all requests.
- Basic spike arrest protection.
- Proxying all traffic to a mock target backend.

---

## 2. Proxy Details

| **Name**               | SimpleSecureProxy                 |
|------------------------|-----------------------------------|
| **Version**            | 1.0                               |
| **Base Path**          | `/v1/secure-api`                  |
| **Description**        | A basic, secure Apigee X proxy with API key authentication and spike arrest applied to all incoming requests. |

---

## 3. Routing Rules

- **Route Rule:**  
  - **Name**: `default`
  - **Condition**: None (all requests)
  - **Target Endpoint:** `default` (as defined below)

---

## 4. Target Endpoints

- **Target Endpoint:** `default`
  - **Target URL:** `https://mocktarget.apigee.net`
  - All requests passing verification and spike arrest are forwarded here.

---

## 5. Policies

### a) VerifyAPIKey (`verify-apikey`)
- **Type:** Security
- **Summary:** Ensures every request contains a valid API key.
- **Configuration:**  
  - **Reference:** `request.queryparam.apikey`

<details>
  <summary>XML</summary>

  ```xml
  <VerifyAPIKey name="verify-apikey">
      <APIKey ref="request.queryparam.apikey"/>
  </VerifyAPIKey>
  ```
</details>

---

### b) SpikeArrest (`spike-arrest`)
- **Type:** Traffic Management
- **Summary:** Limits incoming traffic rate to prevent abuse.
- **Configuration:**
  - **Rate:** `5ps` (5 requests per second)

<details>
  <summary>XML</summary>

  ```xml
  <SpikeArrest name="spike-arrest">
      <Rate>5ps</Rate>
  </SpikeArrest>
  ```
</details>

---

## 6. Generated Files

| **Path**                          | **Type**             | **Purpose**                                 |
|----------------------------------- |----------------------|---------------------------------------------|
| apiproxy/SimpleSecureProxy.xml     | API Proxy Config     | Root descriptor for proxy + endpoints       |
| apiproxy/proxies/default.xml       | Proxy Endpoint       | Defines proxy endpoint, flows, and routing  |
| apiproxy/targets/default.xml       | Target Endpoint      | Defines backend target endpoint             |
| apiproxy/policies/verify-apikey.xml| Policy               | Verifies API key in request                 |
| apiproxy/policies/spike-arrest.xml | Policy               | Limits request rate                         |

---

## 7. Security Design

- **Authentication**:  
  - Implements API Key validation. Requests without a valid API key in the request query parameter `apikey` are rejected.
- **Rate Limiting**:  
  - Enforces a maximum of 5 requests/second per client/app (implementation depends on Apigee key configuration).

---

## 8. Request Flow

1. **Ingress:** Client makes a request to `/v1/secure-api`.
2. **PreFlow (ProxyEndpoint)**:
    - **Step 1:** `verify-apikey` policy checks for a valid API key in the query string.
    - **Step 2:** `spike-arrest` policy applies rate limiting.
3. **Routing:** If passes checks, the request is routed to the target endpoint.
4. **TargetEndpoint (`default`):**  
    - Forwards the request to `https://mocktarget.apigee.net`.
5. **Egress:** Response forwarded back to client.

---

## 9. Assumptions

- API consumers will provide an `apikey` as a query parameter.
- The Apigee environment has at least one valid API product and developer app with key provisioning configured.
- No other methods of authentication will be used (e.g., OAuth, JWT).
- Traffic rate limiting is sufficient at `5ps`; adjust as needed per usage.
- No further path-based routing is required beyond the root base path.

---

## 10. Clarifications Required

_(No explicit clarifications requested by client at this time.)_

If needed, please clarify:
- Should API keys also be accepted via header or other means?
- Should error messages/responses be customized?
- Should policies also apply to response flows?
- Should routing support versioning or path-conditional logic?

---

## 11. Deployment Structure

**Directories & Files:**
```
apiproxy/
├── SimpleSecureProxy.xml
├── policies/
│   ├── spike-arrest.xml
│   └── verify-apikey.xml
├── proxies/
│   └── default.xml
└── targets/
    └── default.xml
```
**Deployment Steps:**
- Import this directory as an Apigee API proxy.
- Deploy to desired environment(s) (e.g., test, prod).
- Ensure developer apps with valid API keys are registered.

---

## 12. Testing Recommendations

### Positive Tests
- **Valid API key:**  
  - Request to `/v1/secure-api?apikey=<valid_key>` → Should receive 200 response from target.
- **Within rate limit:**  
  - Up to 5 requests/second with valid key should succeed.

### Negative Tests
- **Missing/Invalid API key:**  
  - Request to `/v1/secure-api` or with invalid key → Should receive 401 Unauthorized.
- **Rate limit exceeded:**  
  - >5 requests/second with same key → Should receive 429 "Spike Arrest violation".
- **Malformed requests:**  
  - Check error handling for bad/malformed requests.

### Security
- **Replay / Brute-force:**  
  - Attempt rapid-fire with multiple API keys and observe protection.

### Observability
- **Logs:**  
  - Ensure policy failures are logged appropriately.
- **Stats:**  
  - Monitor Apigee dashboard to verify spike arrest triggers as expected.

---

**END OF DOCUMENTATION**
```