```markdown
# GetApigeeSecureProxy Documentation

## 1. Requirement Summary

This Apigee X proxy, `GetApigeeSecureProxy`, is designed to:
- Expose a secured GET API at `/v1/apigee-secure-api`.
- Enforce API key-based authentication (via `x-api-key` header).
- Apply spike arrest to cap the request rate to 5 requests per second.
- Forward authorized, rate-limited requests to a backend target.

---

## 2. Proxy Details

- **Name:** GetApigeeSecureProxy
- **Version:** 1.0
- **Base Path:** `/v1/apigee-secure-api`
- **Description:** A basic Apigee X proxy enforcing API key authentication and spike arrest for traffic smoothing.
- **HTTP Methods Supported:** GET

---

## 3. Routing Rules

- **RouteRule:** `default`
  - **Condition:** (none, applies to all requests matching the base path)
  - **Target:** `default` (see Target Endpoints section)
  - **ProxyEndpoint:** `/v1/apigee-secure-api`

---

## 4. Target Endpoints

### Target: `default`

- **Target URL:** `https://mocktarget.apigee.net`
- **Connection Type:** HTTP(S)
- **Pre/Post Flows:** None customized, default pass-through for requests and responses.
- **Purpose:** Acts as a mock backend for demonstrating proxy flow.

---

## 5. Policies

### a. VerifyAPIKey: `verify-apikey`
- **Type:** VerifyAPIKey
- **Enforcement:** Validates presence of a valid API key.
- **Source:** HTTP header `x-api-key`
- **Success:** Request proceeds if key is valid.
- **Failure:** 401 Unauthorized if key is missing/invalid.

### b. SpikeArrest: `spike-arrest`
- **Type:** SpikeArrest
- **Rate:** 5 requests per second
- **Purpose:** Smooths out traffic spikes, protecting backend.

**Policy Attachment:**  
Both policies are attached to the **PreFlow** `Request` segment of the proxy endpoint.

---

## 6. Generated Files

| Path                                    | Type           | Description                                      |
|------------------------------------------|----------------|--------------------------------------------------|
| apiproxy/GetApigeeSecureProxy.xml        | api_proxy      | API proxy configuration                          |
| apiproxy/proxies/default.xml             | proxy_endpoint | Proxy endpoint with flows/rules/config            |
| apiproxy/targets/default.xml             | target_endpoint| Target endpoint configuration                    |
| apiproxy/policies/verify-apikey.xml      | policy         | API key verification policy definition           |
| apiproxy/policies/spike-arrest.xml       | policy         | Spike arrest/flood control policy definition     |

---

## 7. Security Design

- **API Key Verification:** Enforced for every request via `VerifyAPIKey` policy. The key must be passed in `x-api-key` header.
- **Rate Limiting:** Controlled by `SpikeArrest` policy (5 requests/sec), mitigating denial of service or accidental flooding.
- **No Additional Auth Measures:** There is no OAuth, JWT, or mutual TLS configured.
- **Transport Security:** Target endpoint runs on HTTPS; only secure traffic is forwarded.

---

## 8. Request Flow

1. **Incoming Request:** Client sends a GET request to `/v1/apigee-secure-api`.
2. **PreFlow – Request:**
    - **Step 1:** `verify-apikey` policy checks `x-api-key` header. 
      - If invalid, reject with 401.
    - **Step 2:** `spike-arrest` ensures total requests don't exceed 5/sec.
      - If rate exceeded, respond with 429 error.
3. **Route Rule:** Forwards request to the `default` target.
4. **Target Communication:** Proxies request to `https://mocktarget.apigee.net`.
5. **Return Path:** Backend response is relayed back through proxy (no postprocessing).

---

## 9. Assumptions

- All API consumers will possess and provide a valid API key.
- The backend at `https://mocktarget.apigee.net` is stable and trusted.
- Unauthorized or over-quota requests will be handled by Apigee's standard error handling unless custom policies/errors are added.
- Only GET requests are processed at this endpoint.

---

## 10. Clarifications Required

- **API Key Management:** How are API keys issued, rotated, and revoked? Are there specific key stores/orgs to use?
- **Error Handling:** Are custom error messages needed for authentication or rate limiting failures?
- **Allowed IP Addresses:** Is IP whitelisting/blacklisting required?
- **Response Modifications:** Any need for response augmentation or headers?
- **Payload Validation:** Should any input data validation be enforced?

---

## 11. Deployment Structure

```plaintext
apiproxy/
├── GetApigeeSecureProxy.xml           # API proxy definition
├── policies/
│   ├── spike-arrest.xml               # Spike Arrest policy
│   └── verify-apikey.xml              # API Key verification policy
├── proxies/
│   └── default.xml                    # Proxy endpoint config
├── targets/
│   └── default.xml                    # Target endpoint config
```

---

## 12. Testing Recommendations

### Example Test Request

```http
GET /v1/apigee-secure-api HTTP/1.1
Host: {APIGEE_HOST}
Content-Type: application/json
x-api-key: <API_KEY>
```

### Test Scenarios

1. **Valid API Key, Under Rate Limit**
   - Expect: 200 OK, backend response.

2. **Missing API Key**
   - Omit `x-api-key` header.
   - Expect: 401 Unauthorized.

3. **Invalid API Key**
   - Use a fake/incorrect key.
   - Expect: 401 Unauthorized.

4. **Exceed Spike Arrest Limit**
   - Send >5 requests/sec with the same key/IP.
   - Expect: 429 Too Many Requests.

5. **Backend Unavailable**
   - Simulate target endpoint downtime.
   - Expect: 502 Bad Gateway/504 Gateway Timeout.

6. **Method Not Allowed**
   - Attempt POST/PUT/DELETE.
   - Expect: 405 Method Not Allowed (if enforced upstream).

---

**End of Documentation**
```
