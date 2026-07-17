# TestSecureProxy Documentation

## 1. Requirement Summary

TestSecureProxy implements a secure API proxy enforcing:
- **API key-based authentication**: Only requests with a valid API key are allowed.
- **Spike Arrest**: Limits incoming traffic to smooth bursts and protect backend by restricting the rate of requests.

## 2. Proxy Details

| Attribute       | Value                                           |
|-----------------|------------------------------------------------|
| Name            | `TestSecureProxy`                              |
| Version         | `1.0.0`                                        |
| Base Path       | `/v1/secure-api`                               |
| Description     | API Proxy enforcing API key authentication and spike arrest for basic security and traffic smoothing. |

## 3. Routing Rules

| Name    | Condition | Target Endpoint  |
|---------|-----------|-----------------|
| default | (always)  | default         |

All requests matching the base path are routed to the default target endpoint.

## 4. Target Endpoints

| Name     | Type            | URL                              |
|----------|-----------------|----------------------------------|
| default  | HTTP            | `https://mocktarget.apigee.net`  |

## 5. Policies

### (a) `verify-apikey` (VerifyAPIKey)
- **Type**: API Key Verification
- **Reference**: `request.queryparam.apikey`
- **Purpose**: Validates the API key sent as a query parameter named `apikey`.
- **Behavior**: Rejects requests with missing or invalid API key.

### (b) `spike-arrest` (SpikeArrest)
- **Type**: Spike Arrest (rate limiting)
- **Rate**: 5 requests per second (`5ps`)
- **Purpose**: Smooth out traffic spikes to prevent backend overload.

## 6. Generated Files

| Path                                    | Description                         |
|-----------------------------------------|-------------------------------------|
| apiproxy/TestSecureProxy.xml            | API proxy top-level definition      |
| apiproxy/proxies/default.xml            | ProxyEndpoint configuration         |
| apiproxy/targets/default.xml            | TargetEndpoint configuration        |
| apiproxy/policies/verify-apikey.xml     | VerifyAPIKey policy                 |
| apiproxy/policies/spike-arrest.xml      | SpikeArrest policy                  |

## 7. Security Design

- **Authentication**: Required via API key in `apikey` query parameter.
  - Requests without a valid API key are rejected before hitting the target endpoint.
- **Rate Limiting**: 5 requests per second enforced globally on all incoming traffic.

No OAuth, JWT, IP whitelisting, or mutual TLS specified. Security relies on API key and rate limiting.

## 8. Request Flow

1. **Client sends request** to `/v1/secure-api` (any HTTP method).
2. **PreFlow (Request)**
    - [1] **verify-apikey**: Request must include a valid `apikey` in the query parameters.
    - [2] **spike-arrest**: Only up to 5 requests per second are allowed.
    - If either policy fails, request is rejected with appropriate error.
3. **Routing**: If both policies pass, request is routed to `https://mocktarget.apigee.net`.
4. **Responses**: Flow through as-is; no additional response policies applied.

## 9. Assumptions

- API key management (provisioning, rotation, invalidation) is configured outside this proxy.
- The API key is always expected in the query parameter `apikey` (not in headers or body).
- No per-app/user limiting — `spike-arrest` is applied globally.
- No additional logging, transformation, or header manipulation is performed.
- No custom fault handling specified; default policy behavior applies to errors.

## 10. Clarifications Required

- **API Key Storage/Verification**: Should the API keys be mapped to Apigee developer apps, or are static keys sufficient?
- **Response Handling**: Should any custom error messages or headers be applied for policy rejections?
- **Additional Security**: Are other security mechanisms (CORS, JWT, IP restriction) required?
- **Limiting Scope**: Should spike arrest apply per API key/app, or is global sufficient?
- **API Key in header**: Should header-based API key (`x-api-key`) be supported as well?

## 11. Deployment Structure

```
apiproxy/
├── TestSecureProxy.xml                  # API proxy definition
├── proxies/
│   └── default.xml                      # ProxyEndpoint
├── targets/
│   └── default.xml                      # TargetEndpoint
└── policies/
    ├── verify-apikey.xml                # VerifyAPIKey policy
    └── spike-arrest.xml                 # SpikeArrest policy
```

## 12. Testing Recommendations

- **Happy Path**: Request with a valid API key succeeds.
    - `curl "https://{env-host}/v1/secure-api?apikey={VALID_KEY}"`
- **Missing API Key**: Request is rejected with an authentication error.
- **Invalid API Key**: Request is rejected with an authentication error.
- **Spike Arrest Trigger**: Send >5 requests per second; confirm extra requests return a 429/RateLimit error.
- **Response Pass-Through**: Non-error requests should return the backend's response.
- **Negative Test**: Attempt requests with API key in header/body — expect rejection.
- **Proxy Path**: Requests to invalid paths (`/v1/wrong-api`) should not be accepted.
- **API Key Leakage**: Ensure apikey isn't logged in response or in error messages.
- **Error Codes**: Verify correct HTTP status codes for failures.

---

**End of Documentation**