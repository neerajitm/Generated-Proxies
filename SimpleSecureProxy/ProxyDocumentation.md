# SimpleSecureProxy — Apigee X Proxy Documentation

## 1. Requirement Summary

**SimpleSecureProxy** is an API proxy designed to expose a secure HTTP endpoint at `/v1/secure-api`, protected by API key validation (received via query parameter) and rate-limited to 5 requests per second (spike arrest). The proxy forwards valid requests to a backend mock endpoint.

---

## 2. Proxy Details

| Property         | Value                                   |
|------------------|-----------------------------------------|
| Name             | SimpleSecureProxy                       |
| Version          | 1.0.0                                   |
| Base Path        | /v1/secure-api                          |
| Description      | Basic secure API with API key and spike arrest policies. |

---

## 3. Routing Rules

| Name    | Condition       | Target Endpoint |
|---------|----------------|----------------|
| default | *Always* (no condition) | default        |

- All incoming requests matching `/v1/secure-api/*` are forwarded to the `default` target.

---

## 4. Target Endpoints

| Name    | URL                               |
|---------|-----------------------------------|
| default | https://mocktarget.apigee.net     |

The target endpoint simply forwards proxied requests to the above URL.

---

## 5. Policies

### a. `verify-apikey` (VerifyAPIKey)

- **Type:** VerifyAPIKey
- **Purpose:** Ensures the presence and validity of an API key in the `apikey` query parameter.
- **Configuration:**
  - Extract API key from `request.queryparam.apikey`.

### b. `spike-arrest` (SpikeArrest)

- **Type:** SpikeArrest
- **Purpose:** Limits incoming request traffic to prevent backend overload.
- **Configuration:**
  - Rate: 5 requests per second (`5ps`).

**Policy Attachment:**
- Both policies are attached to the **PreFlow Request** of the proxy endpoint, meaning they apply to all incoming requests.

---

## 6. Generated Files

| File Path                              | Type             | Description                                      |
|----------------------------------------|------------------|--------------------------------------------------|
| apiproxy/SimpleSecureProxy.xml         | API Proxy        | Main API Proxy descriptor                        |
| apiproxy/proxies/default.xml           | Proxy Endpoint   | Defines endpoint flows, routing, and base path    |
| apiproxy/targets/default.xml           | Target Endpoint  | Outbound connection details                      |
| apiproxy/policies/verify-apikey.xml    | Policy           | API Key validation logic                         |
| apiproxy/policies/spike-arrest.xml     | Policy           | Spike arrest configuration                       |

---

## 7. Security Design

- **API Key Required:** Every request must include a valid key in the `apikey` query parameter. Failure to provide or validate it will result in a 401 Unauthorized response.
- **Traffic Throttling:** The `spike-arrest` policy restricts consumers to a maximum of 5 requests per second, offering basic protection against abuse and traffic spikes.
- **Transport Security:** Target endpoint uses HTTPS, ensuring data privacy over the network.

---

## 8. Request Flow

1. **Incoming Request** (`/v1/secure-api`)
    - ↓
2. **PreFlow (Request):**
    - a. **Policy:** `verify-apikey` (Validates API Key)
    - b. **Policy:** `spike-arrest` (Applies rate limiting)
    - ↓
3. **Proxy Routing**
    - Forwards request to target endpoint (`https://mocktarget.apigee.net`)
    - ↓
4. **Target Endpoint**
    - Handles request and returns response
    - ↓
5. **PreFlow (Response):**
    - No policies attached
    - ↓
6. **Client Receives Response**

---

## 9. Assumptions

- Consumers are aware the API key must be provided as a query parameter (?apikey=...).
- Backend does not require additional security policies (e.g., OAuth, JWT).
- The API key verification uses Apigee's built-in mechanism, with keys registered in the Apigee developer/app database.
- Backend (https://mocktarget.apigee.net) is always available and trusted for demo purposes.
- There is no path or method-specific routing—ALL methods/paths on `/v1/secure-api` are handled identically.

---

## 10. Clarifications Required

_N/A (no outstanding clarifications needed at this time, based on provided configuration)._

---

## 11. Deployment Structure

1. **Proxy Bundle Created** — Contains XML files as specified above.
2. **Deployment Target** — Apigee X Environment (assumed to be pre-configured).
3. **Installation/Deployment** — Standard Apigee deployment process: Import proxy; deploy to `test` / `prod` environment.

---

## 12. Testing Recommendations

- **API Key Enforcement:**
  - Send requests with and without `apikey` parameter; expect 401 Unauthorized if absent/invalid.
  - Test with valid API key expected to succeed.
- **Spike Arrest:**
  - Rapidly fire more than 5 requests per second; some should return `429 Too Many Requests`.
- **Endpoint Reachability:**
  - Confirm valid requests are forwarded to `https://mocktarget.apigee.net` and responses are relayed.
- **HTTPS Outbound:** 
  - Inspect that outbound traffic from Apigee to the backend is over HTTPS.
- **Error Handling:**
  - Validate appropriate error responses for all failure cases (missing API key, over limit, etc.).
- **Rate Limiting:** 
  - Test concurrency and rate limit boundaries for corner cases (e.g., 5 requests exactly, bursts, etc.).

---

**Document Version:** 2024-06  
**Proxy Version:** 1.0.0