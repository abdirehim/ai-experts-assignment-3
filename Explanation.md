# Explanation

## What was the bug?

The `Client.request()` method failed to refresh expired or invalid tokens when `oauth2_token` was a dictionary instead of an `OAuth2Token` instance. The test `test_api_request_refreshes_when_token_is_dict` would fail because dict tokens were never refreshed.

## Why did it happen?

The refresh condition used incomplete type checking:
```python
if not self.oauth2_token or (isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired):
```

This only refreshes when the token is `None` OR when it's an `OAuth2Token` AND expired. A dict token is neither `None` nor an `OAuth2Token`, so the condition evaluates to `False` and refresh never occurs.

## Why does your fix actually solve it?

The fix changes the condition to:
```python
if not self.oauth2_token or not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:
```

Now it refreshes when:
1. Token is `None` (missing)
2. Token is NOT an `OAuth2Token` instance (e.g., dict)
3. Token is an `OAuth2Token` AND expired

This ensures all invalid token states trigger a refresh.

## One realistic edge case your tests still don't cover

Concurrent requests with token expiration: If two requests arrive simultaneously and both see an expired token, both will call `refresh_oauth2()` at the same time, potentially causing race conditions or duplicate refresh calls. The code doesn't use locks or atomic operations to prevent this.
