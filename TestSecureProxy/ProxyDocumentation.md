# TestSecureProxy API Proxy Documentation

## 1. Requirement Summary

- **Purpose**: To secure an API endpoint by enforcing API key authentication (via a query parameter) and applying spike arrest to prevent traffic spikes.
- **Authentication**: Requires `apikey` as a query parameter (`request.queryparam.apikey`).
- **Rate Limiting**: Enforces a maximum of 5 requests per second for all incoming requests.
- **Proxy Base Path**: `/v1/secure-api`
- **Target**: Proxies requests to `https://mocktarget.apigee.net`

---

## 2. Proxy Details

- **Name**: `TestSecureProxy`
- **Version**: 1.0.0
- **Base Path**: `/v1/secure-api`
- **Description**: A secure API proxy enforcing API key authentication via request.queryparam.apikey and spike arrest (5 requests/second) on all incoming requests.

---

## 3. Routing Rules

| Name    | Condition | Target Endpoint |
|---------|-----------|----------------|
| default | *Always*  | default        |

- All requests to `/v1/secure-api` are routed to the `default` target endpoint.

---

## 4. Target Endpoints

| Name    | URL                               |
|---------|-----------------------------------|
| default | https://mocktarget.apigee.net     |

---

## 5. Policies

| Name           | Type           | Applied Flow | Configuration                                                |
|----------------|---------------|-------------|--------------------------------------------------------------|
| verify-apikey  | VerifyAPIKey   | PreFlow     | Looks for `apikey` in `request.queryparam.apikey`            |
| spike-arrest   | SpikeArrest    | PreFlow     | Limits requests to `5 requests per second (5ps)`             |

---

## 6. Generated Files

**1. Proxy Configuration**
- `apiproxy/TestSecureProxy.xml`: Declares proxy, endpoints.

**2. Proxy Endpoint**
- `apiproxy/proxies/default.xml`: Proxy endpoint logic, base path, routing, and PreFlow attachments.

**3. Target Endpoint**
- `apiproxy/targets/default.xml`: Points to `https://mocktarget.apigee.net`.

**4. Policies**
- `apiproxy/policies/verify-apikey.xml`: API key verification.
- `apiproxy/policies/spike-arrest.xml`: Spike arrest, 5 req/sec.

---

## 7. Security Design

- **API Key Validation**: 
  - Uses the VerifyAPIKey policy to require a valid key in `request.queryparam.apikey`.
  - Requests without a valid API key are rejected.

- **Spike Arrest Policy**: 
  - Mitigates Denial of Service (DoS) risks by throttling traffic to 5 requests per second globally for the API proxy.

- **No data transformation or other authentication occurs** beyond the API key check.

---

## 8. Request Flow

1. **Inbound Request** hits `/v1/secure-api/*`.
2. **PreFlow (Request) logic** executes:
   - `verify-apikey`: Checks for `apikey` in request query params and validates.
   - `spike-arrest`: Applies rate limiting (5 requests/sec).
3. **Route and Forward**:
   - If policies succeed, routes to the `default` target (`https://mocktarget.apigee.net`).
4. **Response** is returned from the target unmodified.

---

## 9. Assumptions

- All callers must provide a valid `apikey` as a query parameter in every request.
- Spike arrest is sufficient for rate limiting; no additional quotas or analytics policies are required for this version.
- Proxy is not intended to modify, log, or transform request or response bodies/etc.
- The target endpoint (`https://mocktarget.apigee.net`) is only for demonstration/testing purposes.

---

## 10. Clarifications Required

_(No clarifications recorded for this version. Please specify if there are expected changes in authentication source, more granular routing, or advanced security.)_

---

## 11. Deployment Structure

```
apiproxy/
├── TestSecureProxy.xml
├── policies/
│   ├── verify-apikey.xml
│   └── spike-arrest.xml
├── proxies/
│   └── default.xml
├── targets/
│   └── default.xml
```

---

## 12. Testing Recommendations

1. **Positive Test**:
    - Call `/v1/secure-api` endpoint with a **valid** `apikey` query parameter.
    - Verify that you receive a proxied response from `https://mocktarget.apigee.net`.

2. **Negative Test (Missing Key)**:
    - Call the endpoint **without** the `apikey`.
    - Confirm that the request is rejected with a `401 Unauthorized` or `403 Forbidden` response.

3. **Negative Test (Spike Arrest)**:
    - Call the endpoint more than 5 times per second (using a script or load tool) with a valid `apikey`.
    - Observe that excess requests are throttled or receive a `429 Too Many Requests` error.

4. **Boundary Test**:
    - Call exactly 5 times per second; all should succeed.

5. **Invalid Key Test**:
    - Call the endpoint with an **invalid** `apikey`.
    - Verify the proxy rejects the request.

6. **Target Reachability**:
    - Ensure that, when authorized and not throttled, requests successfully reach `https://mocktarget.apigee.net`.

---

**End of documentation.**