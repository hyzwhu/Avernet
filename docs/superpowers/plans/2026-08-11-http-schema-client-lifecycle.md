# HTTP Schema Client Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close one-shot HTTP schema catalog clients while leaving an injected shared client under its owner's lifecycle.

**Architecture:** `HttpSchemaCatalog._refresh_async` selects between two explicit ownership paths. An injected `_client` is borrowed and never closed; otherwise the catalog creates an `httpx.AsyncClient` in an `async with` block and closes it after that refresh. The response-handling logic remains shared so both paths retain identical last-known-good behavior.

**Tech Stack:** Python 3.12+, asyncio, httpx, pytest, unittest.mock, Ruff

## Global Constraints

- Preserve the synchronous `SchemaCatalog.refresh(domain) -> bool` contract.
- Do not change refresh-loop ownership of its shared `httpx.AsyncClient`.
- Preserve last-known-good caching, conditional requests, logging, and timeout behavior.
- Touch only Gateway HTTP schema catalog implementation and contract tests, plus this plan.

---

### Task 1: Distinguish transient and shared HTTP client ownership

**Files:**
- Modify: `src/gateway/src/gateway/community/plugins/schema_catalog/http/_plugin.py`
- Test: `src/gateway/tests/contracts/spi/test_http_schema_catalog.py`

**Interfaces:**
- Consumes: `HttpSchemaCatalog.refresh(domain: str) -> bool`, `httpx.AsyncClient`
- Produces: `_refresh_with_client(domain: str, url: str, client: httpx.AsyncClient) -> bool`, with `_refresh_async` retaining its existing signature

- [ ] **Step 1: Replace the client-construction test with failing ownership tests**

```python
def test_refresh_closes_transient_client() -> None:
    catalog = _new_catalog({"bots": "http://example.com/bots.json"})
    transient = _mock_client(_make_response(200, {"version": 1}))
    transient.__aenter__.return_value = transient

    with patch("httpx.AsyncClient", return_value=transient):
        assert catalog.refresh("bots") is True

    transient.__aexit__.assert_awaited_once()


def test_refresh_does_not_close_shared_client() -> None:
    catalog = _new_catalog({"bots": "http://example.com/bots.json"})
    shared = _mock_client(_make_response(200, {"version": 1}))
    catalog._client = shared

    with patch("httpx.AsyncClient") as client_factory:
        assert catalog.refresh("bots") is True

    client_factory.assert_not_called()
    shared.__aexit__.assert_not_awaited()
```

- [ ] **Step 2: Run the focused tests and verify the transient-client test fails**

Run: `cd src/gateway && uv run pytest -q tests/contracts/spi/test_http_schema_catalog.py`

Expected: the transient-client test fails because the current `_get_client()` result is not entered or closed; existing catalog behavior remains green.

- [ ] **Step 3: Implement explicit client ownership paths**

```python
async def _refresh_async(self, domain: str, url: str) -> bool:
    if self._client is not None:
        return await self._refresh_with_client(domain, url, self._client)
    async with httpx.AsyncClient(timeout=_DEFAULT_TIMEOUT) as client:
        return await self._refresh_with_client(domain, url, client)

async def _refresh_with_client(
    self, domain: str, url: str, client: httpx.AsyncClient
) -> bool:
    headers = _build_conditional_headers(
        self._etags.get(domain), self._last_modified.get(domain)
    )
    try:
        response = await client.get(url, headers=headers)
    except httpx.HTTPError as exc:
        logger.warning("schema refresh failed for %s (%s): %s", domain, url, exc)
        return False

    if response.status_code == 304:
        return True

    if response.status_code < 200 or response.status_code >= 300:
        logger.warning(
            "schema refresh for %s (%s): HTTP %s",
            domain,
            url,
            response.status_code,
        )
        return False

    try:
        parsed = _parse_body(response)
    except Exception as exc:
        logger.warning(
            "schema refresh for %s (%s): parse failed: %s",
            domain,
            url,
            exc,
        )
        return False

    if not isinstance(parsed, dict):
        logger.warning(
            "schema for %s is not a mapping; keeping last known-good", domain
        )
        return False

    _store_conditional_headers(response, domain, self._etags, self._last_modified)
    self._cache[domain] = parsed
    return True
```

Remove `_get_client`; its unmanaged construction path is no longer valid.

- [ ] **Step 4: Run focused contract tests and Ruff**

Run: `cd src/gateway && uv run pytest -q tests/contracts/spi/test_http_schema_catalog.py`

Expected: all tests in the file pass.

Run: `cd src/gateway && uv run ruff check src tests`

Expected: `All checks passed!`

- [ ] **Step 5: Run the Gateway test suite**

Run: `cd src/gateway && uv run pytest -q`

Expected: all Gateway tests pass. If dependency synchronization cannot complete, report the exact blocker without claiming success.

- [ ] **Step 6: Commit the scoped change**

```bash
git add docs/superpowers/plans/2026-08-11-http-schema-client-lifecycle.md \
  src/gateway/src/gateway/community/plugins/schema_catalog/http/_plugin.py \
  src/gateway/tests/contracts/spi/test_http_schema_catalog.py
git commit -m "fix(gateway): close transient schema catalog clients"
```
