# Exploiting an API Endpoint Using Documentation

![Platform](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Type](https://img.shields.io/badge/Type-API%20Testing-orange)
![Status](https://img.shields.io/badge/Status-Completed-success)

Trimming a REST API path in Burp Repeater surfaces an unlinked documentation page, revealing an undocumented `DELETE` method with no authorization check — enough to delete another user's account entirely.

## Techniques Used

- API endpoint enumeration by path trimming
- Burp Suite Repeater
- HTTP method tampering (`PATCH` → `DELETE`)
- Exposed documentation discovery

## Attack Summary

```text
Login wiener:peter --> email-change feature --> PATCH /api/user/wiener
Trim path in Repeater --> /user --> 302 --> exposed API documentation
Documentation reveals DELETE /api/user/{username}
DELETE /api/user/carlos --> carlos deleted --> lab solved
```

## Key Vulnerability

**Missing authorization on a DELETE endpoint, discovered via unlinked API documentation.** The application's UI never exposes account deletion, but the underlying API accepts a `DELETE` request for any username — including ones the authenticated user doesn't own.

## Request Analysis

### Legitimate Request
```http
PATCH /api/user/wiener HTTP/1.1
```

### Attack Request
```http
DELETE /api/user/carlos HTTP/1.1
```

## Why It Works

| Factor | Explanation |
|---|---|
| Unlinked ≠ protected | The API docs weren't in the UI, but were still fully reachable |
| No ownership check | Nothing verifies the deleting user matches the target username |
| Predictable REST pattern | `/api/user/{username}` made targeting another account trivial |
| Backend trusts all methods | `DELETE` worked even though the front-end never used it |

## References

- [PortSwigger — Exploiting an API endpoint using documentation](https://portswigger.net/web-security/api-testing/lab-exploiting-api-endpoint-using-documentation)
- [PortSwigger — API Testing methodology](https://portswigger.net/web-security/api-testing)
