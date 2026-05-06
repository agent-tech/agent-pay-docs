---
order: 2
---

# Error Handling

> Source: https://docs.agent.tech/support/error-codes/

# Error Handling

AgentTech SDKs use typed errors and sentinel values for precise error matching.

## Error Types

### RequestError

Returned for HTTP 4xx/5xx responses from the API.

```
if (err instanceof PayApiError) {
  console.log(`HTTP ${err.statusCode}: ${err.body}`);
}
```

```
var reqErr *pay.RequestError
if errors.As(err, &reqErr) {
    log.Printf("HTTP %d: %s", reqErr.StatusCode, reqErr.Body)
}
```

### ValidationError

Returned when the SDK rejects a request before it reaches the API (e.g. empty intent ID). Wraps a sentinel error for `errors.Is` matching.

```
var valErr *pay.ValidationError
if errors.As(err, &valErr) {
    log.Printf("Invalid input: %s", valErr.Message)
}
```

### UnexpectedError

Wraps unexpected internal errors (JSON marshal failure, request creation, etc.).

```
var unexpErr *pay.UnexpectedError
if errors.As(err, &unexpErr) {
    log.Printf("Unexpected: %v", unexpErr.Err)
}
```

## Sentinel Errors

Use `errors.Is` (Go) or error type checking to match specific validation failures.

| Sentinel | Meaning |
| --- | --- |
| `ErrEmptyBaseURL` | baseURL was empty in NewClient |
| `ErrEmptyIntentID` | intentID was empty |
| `ErrEmptySettleProof` | settleProof was empty in SubmitProof |
| `ErrMissingAuth` | ExecuteIntent called without auth |
| `ErrNilParams` | params argument was nil |

### Example (Go)

```
if errors.Is(err, pay.ErrEmptyIntentID) {
    // intent ID was empty
}
```

## HTTP Status Codes

| Status Code | Meaning |
| --- | --- |
| `400` | Bad request — invalid parameters, amount out of range, malformed input, or out-of-range pagination (`page`, `page_size`) |
| `401` | Unauthorized — missing or invalid credentials |
| `402` | Payment required — agent wallet has insufficient stablecoin balance to satisfy `ExecuteIntent` |
| `403` | Forbidden — reserved for future use. v2 ownership rejections return `404`, not `403` (see below) |
| `404` | Not found — intent does not exist **or** is owned by a different agent. v2 ownership checks return the same `404 payment intent not found` body for both cases, so the endpoint cannot be used to enumerate intent IDs across agents |
| `429` | Rate limited — too many requests (60 req/min/IP typical) |
| `503` | Service unavailable — temporary backend issue (e.g. proxy wallet temporarily out of funds) |

## Rate Limiting

The API allows approximately **60 requests per IP per minute**. On HTTP 429, implement exponential backoff:

```
var reqErr *pay.RequestError
if errors.As(err, &reqErr) && reqErr.StatusCode == 429 {
    time.Sleep(backoff)
    // retry
}
```
