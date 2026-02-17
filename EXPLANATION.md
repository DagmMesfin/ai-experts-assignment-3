## What was the bug?

`Client.request()` ignored OAuth2 tokens stored as plain dicts, so API calls made after persisting/reloading a token never attached an `Authorization` header and always failed authentication. Pytest exposed it immediately: `test_api_request_refreshes_when_token_is_dict` failed because the refresh path never hydrated the dict into an `OAuth2Token`, leaving the header unset, and the follow-up test I added, which was `test_api_request_uses_dict_token_when_valid` failed for the same reason even when the dict token was fresh.

## Why did it happen?

The method only handled `OAuth2Token` instances when checking expirations and building headers. Because the attribute is annotated as `Union[OAuth2Token, Dict, None]`, callers can persist the token as a dict. Once such a dict is assigned back, the code neither converted it nor refreshed it, so `Authorization` remained absent.

## Why does your fix actually solve it?

Before checking expiration we now coerce dict tokens into `OAuth2Token`, storing the hydrated instance back. That unified representation enables the existing expiration logic to run and ensures the bearer header is produced. If the dict's timestamp is stale, the refresh branch runs and supplies a fresh token, so API calls always carry a valid header.

## What’s one realistic case / edge case your tests still don’t cover?

Simultaneous concurrent requests that refresh at the same time—racing threads could both call `refresh_oauth2()` and briefly overwrite each other. The current single-threaded tests don't exercise that scenario.
