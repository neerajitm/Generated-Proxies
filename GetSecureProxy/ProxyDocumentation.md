# GetSecureProxy - Apigee X Proxy Documentation

## 1. Requirement Summary

**Purpose:**  
Expose a secure GET endpoint `/v1/secure-api` requiring API key verification and protected from traffic spikes.

**Key Requirements:**
- Only allow HTTP GET requests.
- Require valid API key in `x-api-key` header.
- Apply spike arrest to limit incoming request rates.
- Route to an external mock target.

---

## 2. Proxy Details

- **Proxy Name:** GetSecureProxy
- **Version/Revision:** 1
- **Base Path:** `/v1/secure-api`
- **Allowed Method(s):** GET
- **Description:** A secure GET proxy with API key verification and spike arrest protection.

---

## 3. Routing Rules

| Route Name | Condition      | Target Endpoint | Description                          |
|:-----------|:--------------|:---------------|:-------------------------------------|
| default    | (always true) | default        | All requests are routed to default.  |

---

## 4. Target Endpoints

| Target Name | Target URL                           | Description                                 |
|:------------|:-------------------------------------|:--------------------------------------------|
| default     | https://mocktarget.apigee.net        | External mock API used as backend service.  |

---

## 5. Policies

### 5.1. verify-apikey

- **Type:** VerifyAPIKey
- **Configuration:**  
  - Location: HTTP header
  - Header Name: x-api-key
- **Purpose:** Ensures requests are authenticated using a provided API key.

### 5.2. spike-arrest

- **Type:** SpikeArrest
- **Configuration:**  
  - Rate: 5 requests per second (`5ps`)
- **Purpose:** Prevents traffic spikes by limiting incoming request velocity.

---

## 6. Generated Files

| File Path                                | Description                    |
|:------------------------------------------|:-------------------------------|
| apiproxy/GetSecureProxy.xml               | API proxy main definition      |
| apiproxy/proxies/default.xml              | Proxy endpoint configuration   |
| apiproxy/targets/default.xml              | Target endpoint configuration  |
| apiproxy/policies/verify-apikey.xml       | API key verification policy    |
| apiproxy/policies/spike-arrest.xml        | Spike arrest policy            |

---

## 7. Security Design

- **API Key Required:** Yes (header `x-api-key`)
- **Rate Limiting:** Spike arrest at 5 requests/second per consumer/app.
- **Transport Security:** Target endpoint uses HTTPS.
- **No other authentication** is present (e.g., OAuth2).

---

## 8. Request Flow

1. **Client sends GET request** to `/v1/secure-api` with `x-api-key` in header.
2. **PreFlow - Request:**
   - `verify-apikey` policy verifies the key.
   - `spike-arrest` policy applies rate limiting.
3. **Route Rule:** Request routed to target endpoint.
4. **Target Endpoint:** Proxy forwards request to `https://mocktarget.apigee.net`.
5. **Response:** Response returned unchanged to client.

---

## 9. Assumptions

- The consumer has a valid API Key provisioned within Apigee.
- Only GET requests are allowed (others will be rejected by Apigee by default).
- Spike arrest is sufficient for rate limiting unless noted otherwise.
- No response/response flow policies are present.
- No request or response transformations are needed.

---

## 10. Clarifications Required

- Should additional error handling or custom error messages be added (e.g., for invalid API key or rate limit exceeded)?
- Are there any usage analytics/reporting requirements?
- Is CORS support required for browser-based clients?
- Should HTTPS be enforced at the proxy level?

---

## 11. Deployment Structure

```
apiproxy/
├── GetSecureProxy.xml
├── proxies/
│   └── default.xml
├── targets/
│   └── default.xml
└── policies/
    ├── verify-apikey.xml
    └── spike-arrest.xml
```

---

## 12. Testing Recommendations

**Basic Test:**
- **Request:**
    - Method: `GET`
    - URL: `/v1/secure-api`
    - Headers: 
        - `Content-Type: application/json`
        - `x-api-key: <API_KEY>`
- **Expected Status:** 200 (if API Key is valid and request volume < 5QPS)

**Negative Tests:**
- Omit API key → expect 401 Unauthorized.
- Use invalid API key → expect 401 Unauthorized.
- Exceed rate limit (burst 6+ requests/sec) → expect 429 Too Many Requests.
- Use methods other than GET → expect 405 Method Not Allowed.

**Other:**
- Confirm correct routing to `https://mocktarget.apigee.net`.
- Confirm no modification to request/response payloads.

---

**End of Documentation**