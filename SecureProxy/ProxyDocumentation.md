# SecureProxy (Version 1) – Apigee X Documentation

## 1. Requirement Summary

A secure API proxy is required to:
- Expose an endpoint at `/v1/secure-api`.
- Enforce API key validation for all requests.
- Protect backend services with basic traffic protection (rate limiting).
- Route validated traffic to a backend target, accessible at https://mocktarget.apigee.net.

## 2. Proxy Details

| Property    | Value                          |
|-------------|-------------------------------|
| **Name**    | SecureProxy                   |
| **Version** | 1                             |
| **Base Path** | `/v1/secure-api`            |
| **Description** | *None specified*          |

## 3. Routing Rules

- All requests to the Proxy's base path (`/v1/secure-api`) are routed to the **default** target endpoint.
- No conditional routing or additional path-based routing is defined.

## 4. Target Endpoints

- **TargetEndpoint Name**: `default`
- **Target URL**: `https://mocktarget.apigee.net`
- **Target Type**: HTTP(s)

## 5. Policies

### 5.1. VerifyAPIKey Policy (`verify-apikey`)
- **Type**: VerifyAPIKey
- **Configuration**:
  - The API key is expected as a query parameter: `apikey`
  - Reference: `request.queryparam.apikey`
- **Placement**: Proxy Endpoint PreFlow (Request)

### 5.2. SpikeArrest Policy (`spike-arrest`)
- **Type**: SpikeArrest (Rate Limiting)
- **Configuration**:
  - Rate: `5ps` (5 requests per second)
- **Placement**: Proxy Endpoint PreFlow (Request)

## 6. Generated Files

| File Path                               | Type             | Description                               |
|-----------------------------------------|------------------|-------------------------------------------|
| apiproxy/SecureProxy.xml                | API Proxy        | Proxy bundle descriptor                   |
| apiproxy/proxies/default.xml            | Proxy Endpoint   | Main Proxy endpoint definition            |
| apiproxy/targets/default.xml            | Target Endpoint  | Connection config for backend target      |
| apiproxy/policies/verify-apikey.xml     | Policy           | API key validation policy                 |
| apiproxy/policies/spike-arrest.xml      | Policy           | Rate limiting policy                      |

## 7. Security Design

- **Authentication**:   Enforced via API Key, expected as a query parameter. Requests without a valid API Key are rejected.
- **Traffic Protection**:  SpikeArrest policy restricts incoming requests to a maximum of 5 requests per second per client/app.
- **No Additional AuthN**: No OAuth or mutual TLS configured.
- **Backend Exposure**: Backend is not directly exposed; all requests route through Apigee.
- **No request/response content filtering or threat protection implemented.**

## 8. Request Flow

1. **Client makes a request to** `/v1/secure-api`.
2. **VerifyAPIKey Policy** checks for the presence and validity of the API key in `queryparam.apikey`.
    - If missing or invalid, the request is rejected (4xx response).
3. **SpikeArrest Policy** limits requests to 5/sec per client. Exceeding this triggers 429 Too Many Requests.
4. If both policies pass, request is routed to `https://mocktarget.apigee.net`.
5. Target response is returned to the client.

## 9. Assumptions

- API consumers are able to pass an API key as a query parameter (`apikey`).
- The backend target at `https://mocktarget.apigee.net` is reliable and reachable from Apigee.
- No additional request/response transformation is needed.
- No environment-specific per-proxy configuration (e.g., different targets or rate limits) is required.
- Rate limiting is global (not per key), unless otherwise configured at the Apigee deployment/platform.

## 10. Clarifications Required

- **Error Handling**: Should responses include custom error messages or codes for policy violations (invalid key, rate limit)?
- **API Key Location**: Should the key also be accepted via headers or only as a query param?
- **SpikeArrest granularity**: Is `5ps` strict per environment/proxy/client, or should it be dynamic?
- **Response Modifications**: Are there plans for response transformations, or does the proxy serve as pass-through?
- **Logging/Analytics**: Any specific logging or header pass-through required?

## 11. Deployment Structure

```
apiproxy/
├── SecureProxy.xml
├── policies/
│   ├── spike-arrest.xml
│   └── verify-apikey.xml
├── proxies/
│   └── default.xml
└── targets/
    └── default.xml
```

- Deployments would typically target environment(s) such as `test`, `prod`, etc., via the Apigee UI, CLI, or CI/CD automation.

## 12. Testing Recommendations

- **API Key Validation**
  - Request **with no API key**: Should return HTTP 401 Unauthorized/403 Forbidden.
  - Request **with invalid API key**: Should return an authentication error.
  - Request **with valid API key**: Should be allowed through.

- **Spike Arrest**
  - Send >5 requests/s with valid API key. Expect HTTP 429 Too Many Requests for excess.
  - Test requests from multiple clients to evaluate shared/spread limit effect.

- **Successful Routing**
  - Request with valid API key and within rate limit should result in proxy forwarding request to `https://mocktarget.apigee.net` and returning corresponding response.

- **Security**
  - Attempt to access without an API key or with malformed requests.
  - Attempt known bad actors’ patterns (e.g., parameter tampering) to ensure no backend leakage.
  
- **Negative scenarios**
  - Backend unavailable: verify proxy surfaces suitable error codes.
  
- **Miscellaneous**
  - Confirm no sensitive data is returned in error messages.
  - Evaluate latency impact of policies.

---

**End of Documentation**